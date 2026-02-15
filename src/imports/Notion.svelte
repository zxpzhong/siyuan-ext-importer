<script lang="ts">
	import 'virtual:uno.css'; // 这一行必须，https://unocss.dev/integrations/vite
	import { KCol, KRow } from '@ikun-ui/grid';
	import { KDivider } from '@ikun-ui/divider';
    import { KButton } from '@ikun-ui/button';
    import { KInput } from '@ikun-ui/input';
    import { WebPickedFile } from '@/libs/filesystem';
    import { readZip, ZipEntryFile } from '@/libs/zip';
	import { Client } from '@siyuan-community/siyuan-sdk';
    import FileInput from '@/FileInput.svelte';
    import { createEventDispatcher } from 'svelte';
    import Ikun from '@/assets/ikun.svelte';
    import { showMessage } from 'siyuan';
    import { upload } from '@/api';

    const dispatch = createEventDispatcher();
    const client = new Client();

    let current = 0;
    let total = 100;
    let clickImportLoading = false;
    let files;
    let githubZipUrl = '';

    $: dispatch('progressChange', { current, total });

    export let currentNotebook: any = { name: '' };
    export let pluginInstance;

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

    function findCommonRootPrefix(entries: ZipEntryFile[]) {
        const segments = entries.map((entry) => decodePath(entry.filepath).split('/').filter(Boolean));
        if (!segments.length || segments.some((parts) => parts.length === 0)) {
            return '';
        }
        const first = segments[0][0];
        const same = segments.every((parts) => parts[0] === first && parts.length > 1);
        return same ? `${first}/` : '';
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

    function toSiyuanDocPath(filepath: string, pagesPrefix: string, rootPrefix: string) {
        let normalized = decodePath(filepath);
        if (pagesPrefix && normalized.startsWith(pagesPrefix)) {
            normalized = normalized.slice(pagesPrefix.length);
        } else if (rootPrefix && normalized.startsWith(rootPrefix)) {
            normalized = normalized.slice(rootPrefix.length);
        }

        const parts = normalized.split('/').filter(Boolean);
        const fileName = parts.pop() ?? 'untitled.md';
        const title = sanitizeDocSegment(fileName.replace(/\.[^.]+$/, ''));
        const parent = parts.map((segment) => sanitizeDocSegment(segment)).filter(Boolean).join('/');
        return parent ? `/${parent}/${title}` : `/${title}`;
    }

    function removeKnownPrefix(filepath: string, prefixes: string[]) {
        for (const prefix of prefixes) {
            if (prefix && filepath.startsWith(prefix)) {
                return filepath.slice(prefix.length);
            }
        }
        return filepath;
    }

    function makeAttachmentLookupKeys(filepath: string, rootPrefix: string, pagesPrefix: string) {
        const normalized = normalizePath(filepath);
        const decoded = decodePath(filepath);
        const keys = [normalized, decoded];

        const strippedNormalized = removeKnownPrefix(normalized, [pagesPrefix, rootPrefix]);
        const strippedDecoded = removeKnownPrefix(decoded, [pagesPrefix, rootPrefix]);
        keys.push(strippedNormalized, strippedDecoded);

        return Array.from(new Set(keys.filter(Boolean)));
    }

    function extractMarkdownLinkTargets(markdown: string) {
        const links: string[] = [];
        const regex = /\[[^\]]*\]\(([^)\s]+\.md(?:[#?][^)]*)?)\)/g;
        let match: RegExpExecArray | null;
        while ((match = regex.exec(markdown)) !== null) {
            const { path } = splitLink(match[1]);
            links.push(path);
        }
        return links;
    }

    function isLikelyFolderIndexMarkdown(markdown: string, linkedDocCount: number) {
        if (linkedDocCount < 2) {
            return false;
        }
        const compact = markdown
            .replace(/\[[^\]]*\]\([^)]+\.md(?:[#?][^)]*)?\)/g, '')
            .replace(/[#>*\-\s`~]+/g, '')
            .trim();
        return compact.length <= 80;
    }

    function buildVirtualHierarchy(
        markdownOrder: string[],
        markdownByPath: Map<string, string>,
        pagesPrefix: string,
        rootPrefix: string,
    ) {
        const parentMap = new Map<string, string>();
        const markdownPathSet = new Set(markdownByPath.keys());

        for (const markdownPath of markdownOrder) {
            const markdownText = markdownByPath.get(markdownPath);
            if (!markdownText) {
                continue;
            }
            const rawTargets = extractMarkdownLinkTargets(markdownText);
            const resolvedTargets = rawTargets
                .map((target) => resolveRelativePath(markdownPath, target))
                .map((target) => decodePath(target));
            const childTargets = Array.from(new Set(resolvedTargets.filter((target) => markdownPathSet.has(target) && target !== markdownPath)));
            if (!isLikelyFolderIndexMarkdown(markdownText, childTargets.length)) {
                continue;
            }
            for (const child of childTargets) {
                if (!parentMap.has(child)) {
                    parentMap.set(child, markdownPath);
                }
            }
        }

        const docPathMap = new Map<string, string>();
        const titleMap = new Map<string, string>();
        for (const markdownPath of markdownOrder) {
            const normalized = removeKnownPrefix(markdownPath, [pagesPrefix, rootPrefix]);
            const parts = normalized.split('/').filter(Boolean);
            const fileName = parts.pop() ?? 'untitled.md';
            const title = sanitizeDocSegment(fileName.replace(/\.[^.]+$/, ''));
            titleMap.set(markdownPath, title);
        }

        const getDocPath = (markdownPath: string, visited = new Set<string>()): string => {
            const cached = docPathMap.get(markdownPath);
            if (cached) {
                return cached;
            }
            const selfTitle = titleMap.get(markdownPath) || 'untitled';
            if (visited.has(markdownPath)) {
                const fallback = toSiyuanDocPath(markdownPath, pagesPrefix, rootPrefix);
                docPathMap.set(markdownPath, fallback);
                return fallback;
            }
            const parent = parentMap.get(markdownPath);
            if (!parent) {
                const fallback = toSiyuanDocPath(markdownPath, pagesPrefix, rootPrefix);
                docPathMap.set(markdownPath, fallback);
                return fallback;
            }
            visited.add(markdownPath);
            const parentPath = getDocPath(parent, visited);
            visited.delete(markdownPath);
            const path = `${parentPath}/${selfTitle}`;
            docPathMap.set(markdownPath, path);
            return path;
        };

        for (const markdownPath of markdownOrder) {
            getDocPath(markdownPath);
        }
        return docPathMap;
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

    async function downloadZipFromUrl(url: string) {
        const response = await fetch(url);
        if (!response.ok) {
            throw new Error(`下载失败: ${response.status}`);
        }
        const zipBlob = await response.blob();
        const pathName = new URL(url).pathname;
        const fileName = pathName.split('/').pop() || 'github.zip';
        return new WebPickedFile(new File([zipBlob], fileName, { type: 'application/zip' }));
    }

    async function importZipFile(zipFile: WebPickedFile, fromGithubUrl = false) {
        const entries = await listZipEntries(zipFile);
        const workspaceExport = isWolaiWorkspaceExport(entries);
        const rootPrefix = fromGithubUrl ? findCommonRootPrefix(entries) : '';
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
            throw new Error('ZIP 中未找到可导入的 Markdown 页面');
        }

        const attachmentEntries = entries.filter((entry) => entry.extension !== 'md');
        total = markdownEntries.length + attachmentEntries.length;
        current = 0;
        dispatch('startImport');

        const attachmentMap = new Map<string, string>();
        const markdownTextByPath = new Map<string, string>();
        const sortedMarkdownEntries = markdownEntries.sort((a, b) => a.filepath.localeCompare(b.filepath));

        for (const markdownFile of sortedMarkdownEntries) {
            const markdownPath = decodePath(markdownFile.filepath);
            try {
                markdownTextByPath.set(markdownPath, await markdownFile.readText());
            } catch (error) {
                console.error('markdown pre-read failed', markdownFile.filepath, error);
            }
        }

        const hierarchyDocPathMap = workspaceExport
            ? buildVirtualHierarchy(
                sortedMarkdownEntries.map((entry) => decodePath(entry.filepath)),
                markdownTextByPath,
                pagesPrefix,
                rootPrefix,
            )
            : new Map<string, string>();

        for (const attachment of attachmentEntries) {
            try {
                const data = await attachment.readBlob();
                const safeName = sanitizeAsciiSegment(attachment.name);
                const uploadRes = await upload('assets', [new File([data], safeName)]);
                const uploadedPath = uploadRes.succMap?.[safeName];
                if (uploadedPath) {
                    const lookupKeys = makeAttachmentLookupKeys(attachment.filepath, rootPrefix, pagesPrefix);
                    for (const key of lookupKeys) {
                        attachmentMap.set(key, uploadedPath);
                    }
                }
            } catch (error) {
                console.error('attachment import failed', attachment.filepath, error);
            }
            current += 1;
        }

        for (const markdownFile of sortedMarkdownEntries) {
            try {
                const normalizedMarkdownPath = decodePath(markdownFile.filepath);
                const markdownText = markdownTextByPath.get(normalizedMarkdownPath) || await markdownFile.readText();
                const markdownWithAssets = rewriteAttachmentLinks(markdownText, normalizedMarkdownPath, attachmentMap);
                const resCreateDoc = await client.createDocWithMd({
                    markdown: markdownWithAssets,
                    notebook: currentNotebook.id,
                    path: hierarchyDocPathMap.get(normalizedMarkdownPath)
                        || toSiyuanDocPath(markdownFile.filepath, pagesPrefix, rootPrefix),
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
            const importTargets: { file: WebPickedFile, fromGithubUrl: boolean }[] = [];
            for (const file of files || []) {
                const pickedFile = new WebPickedFile(file);
                if (pickedFile.extension === 'zip') {
                    importTargets.push({ file: pickedFile, fromGithubUrl: false });
                }
            }

            if (githubZipUrl.trim()) {
                showMessage(pluginInstance.i18n.downloadingGithubZip || 'Downloading GitHub ZIP...', 5000, 'info');
                const githubZipFile = await downloadZipFromUrl(githubZipUrl.trim());
                importTargets.push({ file: githubZipFile, fromGithubUrl: true });
            }

            if (importTargets.length === 0) {
                throw new Error(pluginInstance.i18n.pleaseSelectZipOrUrl || '请选择 ZIP 文件或填写 GitHub ZIP 链接');
            }

            showMessage(pluginInstance.i18n.startCollectFilePreImport, 1000 * 30, 'info');
            for (const item of importTargets) {
                showMessage(pluginInstance.i18n.detectWolaiZip, 4000, 'info');
                await importZipFile(item.file, item.fromGithubUrl);
            }
            showMessage(pluginInstance.i18n.importFinish, -1, 'info');
        } catch (e) {
            console.error(e);
            const errMsg = e instanceof Error ? e.message : String(e);
            showMessage(`ZIP 导入失败: ${errMsg}`, 1000 * 10, 'error');
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

    <KRow class="mt-2">
        <KCol span={24}>
            <KInput bind:value={githubZipUrl} placeholder="GitHub ZIP URL (e.g. https://github.com/owner/repo/archive/refs/heads/main.zip)" />
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
