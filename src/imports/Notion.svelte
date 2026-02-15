<script lang="ts">
	import 'virtual:uno.css'; // 这一行必须，https://unocss.dev/integrations/vite
	import { KCol, KRow } from '@ikun-ui/grid';
	import { KDivider } from '@ikun-ui/divider';
    import { KButton } from '@ikun-ui/button';
    import { WebPickedFile } from '@/libs/filesystem';
    import { readZip, ZipEntryFile } from '@/libs/zip';
	import { Client } from '@siyuan-community/siyuan-sdk';
    import FileInput from '@/FileInput.svelte';
    import { createEventDispatcher } from 'svelte';
    import Ikun from '@/assets/ikun.svelte';
    import { showMessage } from 'siyuan';

    const dispatch = createEventDispatcher();

    let current = 0;
    let total = 100;

    $: dispatch('progressChange', { current, total });

    const client = new Client();

    export let currentNotebook: any = { name: '' };
    export let pluginInstance;

    let clickImportLoading = false;
    let files;

    async function listZipEntries(file: WebPickedFile): Promise<ZipEntryFile[]> {
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
        } catch {
            return value;
        }
    }

    function decodePath(filepath: string) {
        return normalizePath(filepath)
            .split('/')
            .map((segment) => safeDecodeURIComponent(segment))
            .join('/');
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
            if (!part || part === '.') continue;
            if (part === '..') {
                merged.pop();
                continue;
            }
            merged.push(part);
        }
        return merged.join('/');
    }

    function sanitizeDocSegment(segment: string) {
        return segment.replace(/[\\/:*?"<>|]/g, '_').trim() || 'untitled';
    }

    function sanitizeAsciiSegment(segment: string) {
        const ascii = segment
            .replace(/[^a-zA-Z0-9._-]/g, '_')
            .replace(/_+/g, '_')
            .replace(/^[_\.]+|[_\.]+$/g, '');
        return ascii || 'file';
    }

    function getSafeAssetFileName(sourcePath: string, index: number) {
        const rawName = decodePath(sourcePath).split('/').filter(Boolean).pop() ?? 'file';
        const extMatch = rawName.match(/\.([a-zA-Z0-9]{1,16})$/);
        const ext = extMatch ? `.${extMatch[1].toLowerCase()}` : '';
        return `asset_${index}${ext}`;
    }

    function rewriteAttachmentLinks(markdown: string, filePath: string, attachmentMap: Map<string, string>) {
        const isExternalLink = (value: string) => /^(https?:|data:|mailto:|#)/i.test(value);
        const rewrite = (rawLink: string) => {
            if (isExternalLink(rawLink)) {
                return rawLink;
            }
            const { path, suffix } = splitLink(rawLink);
            const resolvedPath = resolveRelativePath(filePath, path);
            const mapped = attachmentMap.get(resolvedPath) || attachmentMap.get(decodePath(resolvedPath));
            return mapped ? `${mapped}${suffix}` : rawLink;
        };

        return markdown
            .replace(/(!?\[[^\]]*\]\()([^\)\s]+)(\))/g, (_, before, link, after) => `${before}${rewrite(link)}${after}`)
            .replace(/(<(?:img|a)[^>]+(?:src|href)=")([^"]+)(")/g, (_, before, link, after) => `${before}${rewrite(link)}${after}`);
    }

    function isWolaiWorkspaceExport(entries: ZipEntryFile[]) {
        return entries.some((entry) => /(^|\/)pages\//.test(normalizePath(entry.filepath)));
    }

    function findPagesPrefix(filepath: string) {
        const normalized = decodePath(filepath);
        const marker = '/pages/';
        if (normalized.startsWith('pages/')) {
            return 'pages/';
        }
        const idx = normalized.indexOf(marker);
        if (idx >= 0) {
            return normalized.slice(0, idx + marker.length);
        }
        return '';
    }

    function toSiyuanDocPath(filepath: string, pagesPrefix: string) {
        let normalized = decodePath(filepath);
        if (pagesPrefix && normalized.startsWith(pagesPrefix)) {
            normalized = normalized.slice(pagesPrefix.length);
        }

        const parts = normalized.split('/').filter(Boolean);
        const fileName = parts.pop() ?? 'untitled.md';
        const title = sanitizeDocSegment(fileName.replace(/\.[^.]+$/, ''));
        const parent = parts.map((segment) => sanitizeDocSegment(segment)).filter(Boolean).join('/');
        return parent ? `/${parent}/${title}` : `/${title}`;
    }

    async function importWolaiWorkspaceZip(zipFile: WebPickedFile) {
        const entries = await listZipEntries(zipFile);
        const workspaceExport = isWolaiWorkspaceExport(entries);
        const pagesPrefix = workspaceExport
            ? (findPagesPrefix(entries.find((entry) => /(^|\/)pages\//.test(normalizePath(entry.filepath)))?.filepath || '') || 'pages/')
            : '';

        const markdownEntries = entries.filter((entry) => {
            if (entry.extension !== 'md') {
                return false;
            }
            if (!workspaceExport) {
                return true;
            }

            const normalizedPath = normalizePath(entry.filepath);
            const inPages = pagesPrefix ? normalizePath(decodePath(entry.filepath)).startsWith(pagesPrefix) : /(^|\/)pages\//.test(normalizedPath);
            return inPages || !entry.parent;
        });
        if (markdownEntries.length === 0) {
            throw new Error('ZIP 中未找到可导入的 Markdown 页面（请确认是 Wolai Markdown 导出）');
        }
        const attachmentEntries = entries.filter((entry) => entry.extension !== 'md');

        total = markdownEntries.length + attachmentEntries.length;
        current = 0;
        dispatch('startImport');

        const zipName = sanitizeAsciiSegment(zipFile.basename || 'wolai');
        const attachmentMap = new Map<string, string>();

        let assetIndex = 1;
        for (const attachment of attachmentEntries) {
            try {
                const normalizedPath = normalizePath(attachment.filepath);
                const decodedPath = decodePath(attachment.filepath);
                const safeFileName = getSafeAssetFileName(decodedPath, assetIndex);
                const assetPath = `/data/assets/wolai-import/${zipName}/${safeFileName}`;
                const data = await attachment.readBlob();
                const resPutFile = await client.putFile({
                    file: new File([data], safeFileName),
                    path: assetPath,
                });
                if (resPutFile.code === 0) {
                    const siyuanAssetPath = assetPath.replace('/data', '');
                    attachmentMap.set(normalizedPath, siyuanAssetPath);
                    attachmentMap.set(decodedPath, siyuanAssetPath);
                    assetIndex += 1;
                } else {
                    console.error(resPutFile.msg, attachment.filepath);
                }
            } catch (error) {
                console.error('attachment import failed', attachment.filepath, error);
            }
            current += 1;
        }

        const sortedMarkdownEntries = markdownEntries.sort((a, b) => a.filepath.localeCompare(b.filepath));
        for (const markdownFile of sortedMarkdownEntries) {
            try {
                const markdownText = await markdownFile.readText();
                const normalizedMarkdownPath = decodePath(markdownFile.filepath);
                const markdownWithAssets = rewriteAttachmentLinks(markdownText, normalizedMarkdownPath, attachmentMap);
                const resCreateDoc = await client.createDocWithMd({
                    markdown: markdownWithAssets,
                    notebook: currentNotebook.id,
                    path: toSiyuanDocPath(markdownFile.filepath, pagesPrefix),
                });
                if (resCreateDoc.code !== 0) {
                    console.error(resCreateDoc.msg, markdownFile.filepath);
                }
            } catch (error) {
                console.error('markdown import failed', markdownFile.filepath, error);
            }
            current += 1;
        }
    }

	async function onClickImport() {
        clickImportLoading = true;
        try {
            for (const file of files) {
                const pickedFile = new WebPickedFile(file);
                if (pickedFile.extension !== 'zip') {
                    continue;
                }
                showMessage(pluginInstance.i18n.startCollectFilePreImport, 1000 * 30, 'info');
                showMessage(pluginInstance.i18n.detectWolaiZip, 4000, 'info');
                await importWolaiWorkspaceZip(pickedFile);
            }
            showMessage(pluginInstance.i18n.importFinish, -1, 'info');
        } catch (e) {
            console.error(e);
            const errMsg = e instanceof Error ? e.message : String(e);
            showMessage(`Wolai ZIP 导入失败: ${errMsg}`, 1000 * 10, 'error');
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
