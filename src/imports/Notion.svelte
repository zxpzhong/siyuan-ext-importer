<script lang="ts">
	import 'virtual:uno.css'; // 这一行必须，https://unocss.dev/integrations/vite
	import { KCol, KRow } from '@ikun-ui/grid';
	import { KDivider } from '@ikun-ui/divider';
    import { KButton } from '@ikun-ui/button';
    import { type PickedFile, WebPickedFile } from '@/libs/filesystem';
    import { NotionResolverInfo } from '@/libs/formats/notion/notion-types';
    import { readZip, ZipEntryFile } from '@/libs/zip';
    import { getNotionId } from '@/libs/formats/notion/notion-utils';
    import { parseFileInfo } from '@/libs/formats/notion/parse-info';
	import { Client } from '@siyuan-community/siyuan-sdk';
    import { readToMarkdown } from '../libs/formats/notion/convert-to-md';
    import FileInput from '@/FileInput.svelte';
    import { createEventDispatcher } from 'svelte';
    import Ikun from '@/assets/ikun.svelte';
    import { showMessage } from 'siyuan';
    import { type NotionFileInfo } from '../libs/formats/notion/notion-types';

    const dispatch = createEventDispatcher();

    let current = 0;
    let total = 100;

    // 监听 current 和 total 的变化
    $: dispatch('progressChange', { current, total });

    const client = new Client();

    export let currentNotebook: any = { name: '' };

    export let pluginInstance;

    let clickImportLoading = false;

	async function processNotionZips(files: PickedFile[], callback: (file: ZipEntryFile) => Promise<void>) {
		for (let zipFile of files) {
			try {
				await readZip(zipFile, async (_, entries) => {
					for (let entry of entries) {

						// throw an error for Notion Markdown exports
						if (entry.extension === 'md' && getNotionId(entry.name)) {
							console.log('Notion Markdown export detected. Please export Notion data to HTML instead.');
							throw new Error('Notion importer uses only HTML exports. Please use the correct format.');
						}

						// Skip databses in CSV format
						if (entry.extension === 'csv' && getNotionId(entry.name)) continue;

						// Skip summary files
						if (entry.name === 'index.html') continue;

						// Only recurse into zip files if they are at the root of the parent zip
						// because users can attach zip files to Notion, and they should be considered
						// attachment files.
						if (entry.extension === 'zip' && entry.parent === '') {
							try {
								await processNotionZips([entry], callback);
							}
							catch (e) {
								console.log(entry.fullpath, e)
							}
						}
						else {
							await callback(entry);
						}
					}
				});
			}
			catch (e) {
				console.log(zipFile.fullpath)
			}
		}
	}

    async function listZipEntries(file: PickedFile): Promise<ZipEntryFile[]> {
        let zipEntries: ZipEntryFile[] = [];
        await readZip(file, async (_, entries) => {
            zipEntries = entries;
        });
        return zipEntries;
    }

    function normalizePath(filepath: string) {
        return filepath.replaceAll('\\', '/').replace(/^\.\//, '');
    }

    function safeDecodeURIComponent(value: string) {
        try {
            return decodeURIComponent(value);
        }
        catch {
            return value;
        }
    }

    function decodePath(filepath: string) {
        return normalizePath(filepath)
            .split('/')
            .map((segment) => safeDecodeURIComponent(segment))
            .join('/');
    }


    function sanitizeDocSegment(segment: string) {
        return segment.replace(/[\/:*?"<>|]/g, '_').trim();
    }

    function sanitizeAssetSegment(segment: string) {
        const sanitized = segment
            .replace(/[\/:*?"<>|]/g, '_')
            .replace(/[\x00-\x1F]/g, '_')
            .trim();
        return sanitized || '_';
    }


    function toSafeAssetRelativePath(filepath: string) {
        const normalized = decodePath(filepath);
        const safeSegments = normalized
            .split('/')
            .filter(Boolean)
            .filter((segment) => segment !== '.' && segment !== '..')
            .map((segment) => sanitizeAssetSegment(segment));
        return safeSegments.join('/');
    }


    function resolveRelativePath(currentFilePath: string, targetPath: string) {
        const base = normalizePath(currentFilePath);
        const target = normalizePath(targetPath);
        if (target.startsWith('/')) {
            return target.slice(1);
        }
        const baseParts = base.split('/').slice(0, -1).filter(Boolean);
        const targetParts = target.split('/');
        const merged = [...baseParts];
        for (const part of targetParts) {
            if (!part || part === '.') {
                continue;
            }
            if (part === '..') {
                merged.pop();
                continue;
            }
            merged.push(part);
        }
        return merged.join('/');
    }

    function splitLink(link: string) {
        const markerIndex = link.search(/[?#]/);
        if (markerIndex === -1) {
            return { path: link, suffix: '' };
        }
        return {
            path: link.slice(0, markerIndex),
            suffix: link.slice(markerIndex),
        };
    }

    function rewriteAttachmentLinks(markdown: string, filePath: string, attachmentMap: Map<string, string>) {
        const isExternalLink = (value: string) => /^(https?:|data:|mailto:|#)/i.test(value);
        const rewrite = (rawLink: string) => {
            if (isExternalLink(rawLink)) {
                return rawLink;
            }
            const { path, suffix } = splitLink(rawLink);
            const resolvedPath = resolveRelativePath(filePath, path);
            const mapped = attachmentMap.get(resolvedPath);
            return mapped ? `${mapped}${suffix}` : rawLink;
        };

        return markdown
            .replace(/(!?\[[^\]]*\]\()([^\)\s]+)(\))/g, (_, before, link, after) => `${before}${rewrite(link)}${after}`)
            .replace(/(<(?:img|a)[^>]+(?:src|href)=")([^"]+)(")/g, (_, before, link, after) => `${before}${rewrite(link)}${after}`);
    }

    function toSiyuanDocPath(filepath: string, wolaiSpaceExport: boolean) {
        let normalized = decodePath(filepath);
        if (wolaiSpaceExport && normalized.startsWith('pages/')) {
            normalized = normalized.slice('pages/'.length);
        }

        const parts = normalized.split('/').filter(Boolean);
        const fileName = parts.pop() ?? 'untitled.md';
        const title = sanitizeDocSegment(fileName.replace(/\.[^.]+$/, '')) || 'untitled';
        const parent = parts.map((segment) => sanitizeDocSegment(segment)).filter(Boolean).join('/');
        return {
            title,
            path: parent ? `/${parent}/${title}` : `/${title}`,
            normalizedPath: normalized,
        };
    }

    function isWolaiSpaceExport(entries: ZipEntryFile[]) {
        const folderSet = new Set(entries.map((entry) => normalizePath(entry.parent).split('/')[0]).filter(Boolean));
        const hasRootIndex = entries.some((entry) => entry.extension === 'md' && !normalizePath(entry.parent));
        return folderSet.has('pages') && hasRootIndex;
    }

    async function importWolaiZip(zipFile: PickedFile, zipEntries?: ZipEntryFile[]) {
        const entries = zipEntries || await listZipEntries(zipFile);
        const wolaiSpaceExport = isWolaiSpaceExport(entries);
        const markdownEntries = entries.filter((entry) => {
            if (entry.extension !== 'md') {
                return false;
            }
            if (!wolaiSpaceExport) {
                return true;
            }

            const normalized = normalizePath(entry.filepath);
            return normalized.startsWith('pages/') || !entry.parent;
        });
        const attachmentEntries = entries.filter((entry) => entry.extension !== 'md');

        total = markdownEntries.length + attachmentEntries.length;
        current = 0;
        dispatch('startImport');

        const zipName = sanitizeAssetSegment(zipFile.basename || 'wolai');
        const attachmentMap = new Map<string, string>();

        for (const attachment of attachmentEntries) {
            const normalizedPath = normalizePath(attachment.filepath);
            const decodedPath = decodePath(attachment.filepath);
            const safeRelativePath = toSafeAssetRelativePath(attachment.filepath);
            if (!safeRelativePath) {
                current += 1;
                continue;
            }
            const assetPath = `/data/assets/wolai-import/${zipName}/${safeRelativePath}`;
            const data = await attachment.read();
            const safeUploadName = sanitizeAssetSegment(attachment.name);
            const resPutFile = await client.putFile({
                file: new File([data], safeUploadName),
                path: assetPath,
            });
            if (resPutFile.code !== 0) {
                console.error(resPutFile.msg);
            } else {
                const siyuanAssetPath = assetPath.replace('/data', '');
                attachmentMap.set(normalizedPath, siyuanAssetPath);
                attachmentMap.set(decodedPath, siyuanAssetPath);
            }
            current += 1;
        }

        const sortedMarkdownEntries = markdownEntries.sort((a, b) => a.filepath.localeCompare(b.filepath));
        for (const markdownFile of sortedMarkdownEntries) {
            const markdownText = await markdownFile.readText();
            const docPathInfo = toSiyuanDocPath(markdownFile.filepath, wolaiSpaceExport);
            const markdownWithAssets = rewriteAttachmentLinks(markdownText, docPathInfo.normalizedPath, attachmentMap);

            const resCreateDoc = await client.createDocWithMd({
                markdown: markdownWithAssets,
                notebook: currentNotebook.id,
                path: docPathInfo.path,
            });
            if (resCreateDoc.code !== 0) {
                console.error(resCreateDoc.msg);
            }
            current += 1;
        }
    }

    async function importNotionZip(zipFile: PickedFile) {
        const info = new NotionResolverInfo('', false);
        let import_files = [zipFile];

        total = 0;
        current = 0;
        let importIsNotStarted = true;
        await processNotionZips(import_files, async (file) => {
            total += 1;
            if (importIsNotStarted) {
                dispatch('startImport');
                console.log('Starting import');
                importIsNotStarted = false;
            }
            try {
                await parseFileInfo(info, file);
            }
            catch (e) {
                console.log('文件搜集 Import skipped', file.fullpath, e)
            }
        });

        total +=  Object.keys(info.idsToFileInfo).length
        console.log('Creating all document...')
        // 首先查找各个文档作为其他文档的parent出现了多少次
        // 如果不存在的则为叶子文档
        let parentCount: Map<string, number> = new Map();
        for (const fileInfo of Object.values(info.idsToFileInfo) as NotionFileInfo[]) {
            fileInfo.parentIds.forEach(pid => {
                parentCount.set(pid, (parentCount.get(pid) || 0)+1)
            })
        }
        // 创建一个空文档先占位，获取到 blockid
        // 广度优先遍历（BFS）创建文档
        //   由于 createDocWithMd 对于相同路径的文档会创建两遍，如果先创建了较深路径的文档，
        //   则后续创建它的父级文档会出现重复名称的文档
        let depth = 0
        let docSeen = 0;
        while (true) {
            for (const [notionID, fileInfo] of Object.entries(info.idsToFileInfo) as [string, NotionFileInfo][]) {
                if (fileInfo.path.split('/').length === depth) {
                    current += 1;
                    docSeen += 1;
                    // 跳过空内容的叶子文档
                    const path = `${info.getPathForFile(fileInfo)}${fileInfo.title}`;
                    if (!fileInfo.hasContent && !parentCount.get(notionID)) {
                        console.log(`"${path}"'s content is blank, create doc skipped`);
                        continue;
                    }
                    const payload = {
                        markdown: '',
                        notebook: currentNotebook.id,
                        path: path,
                    };
                    const resCreateDocWithMd = await client.createDocWithMd(payload);
                    if (resCreateDocWithMd.code !== 0) {
                        console.error(resCreateDocWithMd.msg);
                        continue;
                    }
                    info.idsToFileInfo[notionID].blockID = resCreateDocWithMd.data;
                }
            }
            depth += 1;
            if (docSeen === Object.keys(info.idsToFileInfo).length) {
                break;
            }
        }
        // 写入所有文档内容
        await processNotionZips(import_files, async (file) => {
            current++;
            try {
                if (file.extension === 'html') {
                    // 写入文档和 database
                    const id = getNotionId(file.name);
                    if (!id) {
                        throw new Error('ids not found for ' + file.filepath);
                    }
                    const fileInfo = info.idsToFileInfo[id];
                    if (!fileInfo) {
                        throw new Error('file info not found for ' + file.filepath);
                    }
                    const path = `${info.getPathForFile(fileInfo)}${fileInfo.title}`;
                    if (fileInfo.blockID === '') {
                        console.log(`"${path}"'s blockID is blank, write doc skipped`)
                        return;
                    }
                    // 处理读取 html
                    console.log(`Importing note ${fileInfo.title}`);
                    const markdownInfo = await readToMarkdown(info, file);
                    // 上传 siyuan database 文件
                    for (const av of markdownInfo.attributeViews) {
                        const avJSONString = JSON.stringify(av);
                        const blob = new Blob([avJSONString], { type: 'application/json' });
                        const resPutFile = await client.putFile({
                            'file': new File([blob], 'data.json', { type: 'application/json' }),
                            'path': `/data/storage/av/${av.id}.json`,
                        })
                        if (resPutFile.code !== 0) {
                            console.log(`put attribute view failed: ${resPutFile.msg}`)
                        }
                    }
                    // 更新文档
                    const resUpdateBlock = await client.updateBlock({
                        data: markdownInfo.content,
                        dataType: 'markdown',
                        id: fileInfo.blockID,
                    })
                    if (resUpdateBlock.code !== 0) {
                        console.error(resUpdateBlock.msg);
                        return;
                    }
                    // 更新 database 关联的文档属性 custom-avs（siyuan某些操作会用到这些属性，比如文档的删除同步）
                    for (const av of markdownInfo.attributeViews) {
                        for (const keyValue of av.keyValues) {
                            if (keyValue.key.type !== 'block') {
                                continue
                            }
                            for (const rowValue of keyValue.values) {
                                if (rowValue?.isDetached) {
                                    continue
                                }
                                const resSetBlockAttrs = await client.setBlockAttrs({
                                    attrs: {
                                        'custom-avs': av.id,
                                    },
                                    id: rowValue.block.id
                                })
                                if (resSetBlockAttrs.code !== 0) {
                                    console.log(resUpdateBlock.msg);
                                }
                            }
                        }
                    }
                } else {
                    // 写入附件
                    const attachmentInfo = info.pathsToAttachmentInfo[file.filepath];
                    if (!attachmentInfo) {
                        throw new Error('attachment info not found for ' + file.filepath);
                    }

                    console.log(`Importing attachment ${file.name}`);

                    const data = await file.read();
                    const resPutFile = await client.putFile({
                        'file': new File([data], file.name),
                        'path': attachmentInfo.pathInSiYuanFs,
                    })
                    if (resPutFile.code !== 0) {
                        console.error(resPutFile.msg);
                        return;
                    }
                }
                console.log(`progress ${current}/${total}`)
            }
            catch (e) {
                console.log(file.fullpath, e)
            }
        });
    }

    function isNotionExport(entries: ZipEntryFile[]) {
        return entries.some((entry) => entry.extension === 'html' && !!getNotionId(entry.name));
    }

	let files;

	async function onClickImport() {
        // 点击导入时触发事件
        clickImportLoading = true;
        try {
            for (const file of files) {
                console.log(`${file.name}: ${file.size} bytes`);
                const pickedFile = new WebPickedFile(file);
                if (pickedFile.extension !== 'zip') {
                    console.log(`unsupported file extension: ${pickedFile.extension}`);
                    continue;
                }

                showMessage(pluginInstance.i18n.startCollectFilePreImport, 1000*30, 'info')
                const zipEntries = await listZipEntries(pickedFile);
                if (isNotionExport(zipEntries)) {
                    showMessage(pluginInstance.i18n.detectNotionZip, 4000, 'info')
                    await importNotionZip(pickedFile);
                } else {
                    showMessage(pluginInstance.i18n.detectWolaiZip, 4000, 'info')
                    await importWolaiZip(pickedFile, zipEntries);
                }
            }
            showMessage(pluginInstance.i18n.importFinish, -1, 'info')
        } finally {
            clickImportLoading = false;
        }
	}
</script>


<div>
	<KRow>
		<KCol span={24}><div class="rounded" />
			<FileInput bind:files accept_ext={['.zip']} />
		</KCol>
	</KRow>

	<KDivider />
	
	<KRow>
		<KCol span={24}>
            <KButton type="primary" cls="mx-2 float-right" on:click={onClickImport} disabled={clickImportLoading}>
                {#if clickImportLoading}
                    <Ikun />
                {/if}
                {pluginInstance.i18n.import}
            </KButton>
		</KCol>
	</KRow>
</div>
