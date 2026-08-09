<script lang="ts">
	import { onMount } from "svelte";
	import MarkdownIt from "markdown-it";
	import sanitizeHtml from "sanitize-html";
	import Icon from "@iconify/svelte";

	interface ShuoshuoListItem {
		id: number;
		content: string;
		reply_count: number;
		created_at: string;
		updated_at: string | null;
		latest_reply: string | null;
		latest_reply_nickname: string | null;
		latest_reply_at: string | null;
	}

	interface Reply {
		id: number;
		shuoshuo_id: number;
		nickname: string;
		content: string;
		is_admin: 0 | 1;
		created_at: string;
	}

	interface ShuoshuoDetail extends ShuoshuoListItem {
		replies: Reply[];
	}

	interface DetailState {
		expanded: boolean;
		loading: boolean;
		error: string;
		detail: ShuoshuoDetail | null;
		formError: string;
		formSuccess: string;
		submitting: boolean;
		nickname: string;
		email: string;
		content: string;
	}

	let {
		apiBaseUrl,
		pageSize = 10,
		authorAvatar = "/favicon/favicon.png",
		authorName = "博主",
	}: {
		apiBaseUrl: string;
		pageSize?: number;
		authorAvatar?: string;
		authorName?: string;
	} = $props();

	/* ---------------- markdown 渲染（先渲染再消毒） ---------------- */

	const md = new MarkdownIt({ html: false, linkify: true, breaks: true });
	const defaultLinkOpen =
		md.renderer.rules.link_open ??
		((tokens, idx, options, _env, self) =>
			self.renderToken(tokens, idx, options));
	md.renderer.rules.link_open = (tokens, idx, options, env, self) => {
		tokens[idx].attrSet("target", "_blank");
		tokens[idx].attrSet("rel", "noopener noreferrer");
		return defaultLinkOpen(tokens, idx, options, env, self);
	};

	const SANITIZE_OPTIONS = {
		allowedTags: [...sanitizeHtml.defaults.allowedTags, "img"],
		allowedAttributes: {
			...sanitizeHtml.defaults.allowedAttributes,
			img: ["src", "alt", "title"],
			a: ["href", "target", "rel"],
		},
		allowedSchemes: ["http", "https", "mailto"],
		allowProtocolRelative: false,
	};

	function renderMarkdown(text: string): string {
		return sanitizeHtml(md.render(text || ""), SANITIZE_OPTIONS);
	}

	/* ---------------- 列表状态 ---------------- */

	let items: ShuoshuoListItem[] = $state([]);
	let page = $state(1);
	let total = $state(0);
	let hasMore = $state(false);
	let loading = $state(true);
	let loadError = $state("");
	let detailMap = $state<Record<number, DetailState>>({});

	const totalPages = $derived(Math.max(1, Math.ceil(total / pageSize)));
	const isPlaceholderBase = $derived(
		apiBaseUrl.trim() === "" ||
			/example\.com|REPLACE_WITH/i.test(apiBaseUrl),
	);

	function getDetailState(id: number): DetailState {
		if (!detailMap[id]) {
			detailMap[id] = {
				expanded: false,
				loading: false,
				error: "",
				detail: null,
				formError: "",
				formSuccess: "",
				submitting: false,
				nickname: "",
				email: "",
				content: "",
			};
		}
		return detailMap[id];
	}

	/* ---------------- 请求 ---------------- */

	async function request<T>(
		path: string,
		init: RequestInit = {},
	): Promise<T> {
		let res: Response;
		try {
			res = await fetch(apiBaseUrl.replace(/\/+$/, "") + path, {
				...init,
				headers: {
					"Content-Type": "application/json",
					...(init.headers ?? {}),
				},
			});
		} catch {
			throw new Error("网络错误，无法连接到说说服务，请稍后再试");
		}

		let payload: { code?: number; data?: T; error?: string } | null = null;
		try {
			payload = await res.json();
		} catch {
			// 非 JSON 响应
		}

		if (!res.ok) {
			const status = res.status;
			let msg = payload?.error || `请求失败（HTTP ${status}）`;
			if (status === 429) msg = "操作太频繁了，请稍后再试";
			else if (status === 404) msg = "内容不存在或已删除";
			else if (status === 500) msg = "服务器开小差了，请稍后再试";
			throw new Error(msg);
		}

		if (!payload || payload.code !== 0) {
			throw new Error(payload?.error || "返回数据格式异常");
		}
		return payload.data as T;
	}

	interface ListData {
		list: ShuoshuoListItem[];
		page: number;
		pageSize: number;
		total: number;
		has_more: boolean;
	}

	async function loadList(targetPage: number) {
		loading = true;
		loadError = "";
		try {
			const data = await request<ListData>(
				`/api/shuoshuo?page=${targetPage}&pageSize=${pageSize}`,
			);
			items = data.list;
			page = data.page;
			total = data.total;
			hasMore = data.has_more;
			// 在事件回调中预建每条说说的展开/表单状态，模板里只做读取
			for (const item of data.list) {
				getDetailState(item.id);
			}
		} catch (e) {
			loadError = e instanceof Error ? e.message : String(e);
		} finally {
			loading = false;
		}
	}

	async function toggleDetail(id: number) {
		const st = getDetailState(id);
		st.expanded = !st.expanded;
		if (st.expanded && !st.detail && !st.loading) {
			st.loading = true;
			st.error = "";
			try {
				st.detail = await request<ShuoshuoDetail>(`/api/shuoshuo/${id}`);
			} catch (e) {
				st.error = e instanceof Error ? e.message : String(e);
			} finally {
				st.loading = false;
			}
		}
	}

	async function submitReply(id: number) {
		const st = getDetailState(id);
		if (!st.nickname.trim() || !st.email.trim() || !st.content.trim()) {
			st.formError = "请填写昵称、邮箱和评论内容";
			return;
		}
		st.submitting = true;
		st.formError = "";
		st.formSuccess = "";
		try {
			const reply = await request<Reply>(`/api/shuoshuo/${id}/replies`, {
				method: "POST",
				body: JSON.stringify({
					nickname: st.nickname.trim(),
					email: st.email.trim(),
					content: st.content.trim(),
				}),
			});

			// 更新详情中的回复列表
			if (st.detail) {
				st.detail = {
					...st.detail,
					replies: [...st.detail.replies, reply],
					reply_count: st.detail.reply_count + 1,
					updated_at: reply.created_at,
					latest_reply: reply.content,
					latest_reply_nickname: reply.nickname,
					latest_reply_at: reply.created_at,
				};
			}

			// 同步更新列表项的最新回复预览
			const item = items.find((i) => i.id === id);
			if (item) {
				item.reply_count += 1;
				item.latest_reply = reply.content;
				item.latest_reply_nickname = reply.nickname;
				item.latest_reply_at = reply.created_at;
				item.updated_at = reply.created_at;
			}

			st.nickname = "";
			st.email = "";
			st.content = "";
			st.formSuccess = "评论发表成功 🎉";
			setTimeout(() => {
				st.formSuccess = "";
			}, 4000);
		} catch (e) {
			st.formError = e instanceof Error ? e.message : String(e);
		} finally {
			st.submitting = false;
		}
	}

	/* ---------------- 分页 ---------------- */

	const pages = $derived.by(() => {
		const current = page;
		const last = totalPages;
		if (last <= 7) {
			return Array.from({ length: last }, (_, i) => i + 1);
		}
		const start = Math.max(2, current - 2);
		const end = Math.min(last - 1, current + 2);
		const result: (number | "…")[] = [1];
		if (start > 2) result.push("…");
		for (let p = start; p <= end; p++) result.push(p);
		if (end < last - 1) result.push("…");
		result.push(last);
		return result;
	});

	/* ---------------- 时间格式 ---------------- */

	function fmtTime(s: string | null | undefined): string {
		if (!s) return "";
		const d = new Date(s.replace(" ", "T") + "Z");
		if (Number.isNaN(d.getTime())) return s;
		return d.toLocaleString("zh-CN", {
			year: "numeric",
			month: "2-digit",
			day: "2-digit",
			hour: "2-digit",
			minute: "2-digit",
			hour12: false,
		});
	}

	function handleAvatarError(event: Event) {
		const img = event.currentTarget as HTMLImageElement;
		img.style.display = "none";
		const fallback = img.nextElementSibling as HTMLElement | null;
		if (fallback) fallback.style.display = "flex";
	}

	onMount(() => {
		loadList(1);
	});
</script>

<div class="shuoshuo text-90">
	{#if isPlaceholderBase}
		<div class="empty-state">
			<span class="empty-icon">
				<Icon icon="material-symbols:link-off-rounded"></Icon>
			</span>
			<p class="empty-title">还没有配置说说 API 地址</p>
			<p class="empty-hint">
				请在 <code>src/config.ts</code> 的 <code>shuoshuoConfig.apiBaseUrl</code>
				中填写实际部署的 Worker 地址。
			</p>
		</div>
	{:else if loading && items.length === 0}
		<div class="empty-state">
			<span class="empty-icon spin">
				<Icon icon="material-symbols:progress-activity-rounded"></Icon>
			</span>
			<p class="empty-title">加载中…</p>
		</div>
	{:else if loadError && items.length === 0}
		<div class="empty-state">
			<span class="empty-icon">
				<Icon icon="material-symbols:cloud-off-rounded"></Icon>
			</span>
			<p class="empty-title">{loadError}</p>
			<button class="btn-regular retry-btn" onclick={() => loadList(1)}>
				重试
			</button>
		</div>
	{:else if items.length === 0}
		<div class="empty-state">
			<span class="empty-icon">
				<Icon icon="material-symbols:forum-outline-rounded"></Icon>
			</span>
			<p class="empty-title">还没有说说，敬请期待～</p>
		</div>
	{:else}
		<div class="shuoshuo-list">
			{#each items as item (item.id)}
				{@const st = detailMap[item.id]}
				<article class="shuoshuo-card">
					<header class="card-header">
						<div class="avatar-wrap">
							<img
								src={authorAvatar}
								alt={authorName}
								loading="lazy"
								onerror={handleAvatarError}
							/>
							<span class="avatar-fallback" aria-hidden="true">
								{authorName.slice(0, 1)}
							</span>
						</div>
						<div class="header-info">
					<span class="author-name">{authorName}</span>
							<time class="author-time" datetime={item.created_at}>
								{fmtTime(item.created_at)}
							</time>
						</div>
					</header>

					<div class="shuoshuo-md md-content text-90">
						{@html renderMarkdown(item.content)}
					</div>

					{#if item.latest_reply && item.reply_count > 0}
						<div class="latest-reply">
							<div class="latest-reply-head">
								<span class="latest-label">最新评论</span>
								<span class="latest-meta">
									{item.latest_reply_nickname} · {fmtTime(
										item.latest_reply_at,
									)}
								</span>
							</div>
							<div class="shuoshuo-md latest-body text-75">
								{@html renderMarkdown(item.latest_reply)}
							</div>
						</div>
					{/if}

					<footer class="card-footer">
						<button
							class="btn-plain reply-toggle"
							onclick={() => toggleDetail(item.id)}
							aria-expanded={st.expanded}
						>
							<span class="toggle-icon">
								<Icon
									icon={
										st.expanded
											? "material-symbols:expand-less-rounded"
											: "material-symbols:forum-outline-rounded"
									}
								></Icon>
							</span>
							<span>{st.expanded ? "收起评论" : "评论"}</span>
							{#if item.reply_count > 0}
								<span class="count-badge">{item.reply_count}</span>
							{/if}
						</button>
						{#if loadError}
							<span class="list-error">{loadError}</span>
						{/if}
					</footer>

					{#if st.expanded}
						<div class="reply-panel">
							{#if st.loading}
								<div class="inline-hint">
									<span class="spin">
										<Icon
											icon="material-symbols:progress-activity-rounded"
										></Icon>
									</span>
									加载评论中…
								</div>
							{:else if st.error}
								<div class="inline-hint error">{st.error}</div>
							{:else if st.detail}
								{#if st.detail.replies.length > 0}
									<ul class="reply-list">
										{#each st.detail.replies as reply (reply.id)}
											<li class="reply-item">
												<div class="reply-head">
													<span class="reply-nickname">
														{reply.nickname}
													</span>
													{#if reply.is_admin === 1}
														<span class="admin-badge">博主</span>
													{/if}
													<time
														class="reply-time"
														datetime={reply.created_at}
													>
														{fmtTime(reply.created_at)}
													</time>
												</div>
												<div class="shuoshuo-md reply-content text-75">
													{@html renderMarkdown(reply.content)}
												</div>
											</li>
										{/each}
									</ul>
								{:else}
									<p class="inline-hint">还没有评论，来抢沙发～</p>
								{/if}

								<form
									class="reply-form"
									onsubmit={(e) => {
										e.preventDefault();
										submitReply(item.id);
									}}
								>
									<div class="form-row">
										<input
											type="text"
											placeholder="昵称"
											maxlength="30"
											autocomplete="nickname"
											bind:value={st.nickname}
										/>
										<input
											type="email"
											placeholder="邮箱（不会公开）"
											maxlength="100"
											autocomplete="email"
											bind:value={st.email}
										/>
									</div>
									<textarea
										rows="3"
										maxlength="2000"
										placeholder="说点什么吧…（支持 Markdown，最多 2000 字）"
										bind:value={st.content}
									></textarea>
									{#if st.formError}
										<p class="form-error">{st.formError}</p>
									{/if}
									{#if st.formSuccess}
										<p class="form-success">{st.formSuccess}</p>
									{/if}
									<div class="form-actions">
										<button
											type="submit"
											class="btn-regular submit-btn"
											disabled={st.submitting}
										>
											{#if st.submitting}
												<span class="spin">
													<Icon
														icon="material-symbols:progress-activity-rounded"
													></Icon>
												</span>
											{/if}
											{st.submitting ? "提交中…" : "发表评论"}
										</button>
									</div>
								</form>
							{/if}
						</div>
					{/if}
				</article>
			{/each}
		</div>

		{#if totalPages > 1}
			<nav class="pagination" aria-label="分页">
				<button
					class="page-btn"
					disabled={page <= 1}
					aria-label="上一页"
					onclick={() => loadList(page - 1)}
				>
					<Icon icon="material-symbols:chevron-left-rounded"></Icon>
				</button>
				{#each pages as p, i (p === "…" ? `ellipsis-${i}` : p)}
					{#if p === "…"}
						<span class="page-btn ellipsis">…</span>
					{:else}
						<button
							class:current={p === page}
							class="page-btn"
							aria-label={`第 ${p} 页`}
							aria-current={p === page ? "page" : undefined}
							onclick={() => loadList(p)}
						>
							{p}
						</button>
					{/if}
				{/each}
				<button
					class="page-btn"
					disabled={!hasMore}
					aria-label="下一页"
					onclick={() => loadList(page + 1)}
				>
					<Icon icon="material-symbols:chevron-right-rounded"></Icon>
				</button>
			</nav>
		{/if}
	{/if}
</div>

<style>
	.shuoshuo-list {
		display: flex;
		flex-direction: column;
		gap: 0.25rem;
	}

	.shuoshuo-card {
		background: var(--license-block-bg);
		border-radius: var(--radius-large);
		padding: 1.1rem 1.5rem;
		margin: 0.5rem 0;
		transition:
			background-color 0.15s ease,
			border-color 0.15s ease;
	}

	.card-header {
		display: flex;
		align-items: center;
		gap: 0.75rem;
		margin-bottom: 0.75rem;
	}

	.avatar-wrap {
		width: 2.25rem;
		height: 2.25rem;
		flex-shrink: 0;
		position: relative;
		border-radius: 50%;
		overflow: hidden;
		background: var(--primary);
	}

	.avatar-wrap img {
		width: 100%;
		height: 100%;
		object-fit: cover;
		display: block;
	}

	.avatar-fallback {
		display: none;
		align-items: center;
		justify-content: center;
		width: 100%;
		height: 100%;
		position: absolute;
		inset: 0;
		color: #fff;
		font-weight: 700;
		font-size: 1rem;
	}

	.header-info {
		display: flex;
		flex-direction: column;
		gap: 0.125rem;
		min-width: 0;
	}

	.author-name {
		font-weight: 700;
		font-size: 0.95rem;
		line-height: 1.4;
	}

	.author-time {
		font-size: 0.75rem;
		opacity: 0.55;
	}

	.md-content {
		margin-bottom: 0.5rem;
	}

	.latest-reply {
		border: 1px solid var(--line-color);
		border-radius: 0.625rem;
		padding: 0.5rem 0.875rem;
		margin: 0.5rem 0 0.25rem;
	}

	.latest-reply-head {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.75rem;
		margin-bottom: 0.125rem;
	}

	.latest-label {
		font-size: 0.75rem;
		font-weight: 600;
		color: var(--btn-content);
	}

	.latest-meta {
		font-size: 0.7rem;
		opacity: 0.55;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.latest-body {
		font-size: 0.85rem;
	}

	.card-footer {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.75rem;
		margin-top: 0.5rem;
	}

	.reply-toggle {
		gap: 0.375rem;
		padding: 0.375rem 0.625rem;
		border-radius: 0.5rem;
		font-size: 0.875rem;
		font-weight: 500;
	}

	.toggle-icon {
		font-size: 1.05rem;
	}

	.count-badge {
		min-width: 1.25rem;
		height: 1.25rem;
		padding: 0 0.375rem;
		border-radius: 999px;
		display: inline-flex;
		align-items: center;
		justify-content: center;
		background: var(--btn-regular-bg);
		color: var(--btn-content);
		font-size: 0.7rem;
		font-weight: 700;
	}

	.list-error {
		font-size: 0.75rem;
		color: #dc2626;
	}

	.reply-panel {
		margin-top: 0.75rem;
		padding-top: 0.75rem;
		border-top: 1px dashed var(--line-color);
	}

	.reply-list {
		display: flex;
		flex-direction: column;
		gap: 0.875rem;
		list-style: none;
		margin: 0 0 1rem;
		padding: 0;
	}

	.reply-item {
		padding-left: 0.875rem;
		border-left: 2px solid var(--line-color);
	}

	.reply-head {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		margin-bottom: 0.25rem;
	}

	.reply-nickname {
		font-weight: 600;
		font-size: 0.875rem;
		color: var(--btn-content);
	}

	.admin-badge {
		background: var(--primary);
		color: #fff;
		font-size: 0.65rem;
		font-weight: 700;
		padding: 0.05rem 0.45rem;
		border-radius: 999px;
		line-height: 1.3;
	}

	:global(html.dark) .admin-badge {
		color: rgba(0, 0, 0, 0.72);
	}

	.reply-time {
		font-size: 0.7rem;
		opacity: 0.5;
		margin-left: auto;
		white-space: nowrap;
	}

	.reply-content {
		font-size: 0.875rem;
	}

	.inline-hint {
		display: flex;
		align-items: center;
		gap: 0.375rem;
		font-size: 0.85rem;
		opacity: 0.65;
		padding: 0.375rem 0;
	}

	.inline-hint.error {
		color: #dc2626;
		opacity: 1;
	}

	.reply-form {
		display: flex;
		flex-direction: column;
		gap: 0.625rem;
	}

	.form-row {
		display: grid;
		grid-template-columns: 1fr 1fr;
		gap: 0.625rem;
	}

	.reply-form input,
	.reply-form textarea {
		width: 100%;
		background: var(--card-bg);
		border: 1px solid var(--line-color);
		border-radius: 0.5rem;
		padding: 0.5rem 0.75rem;
		font-size: 0.875rem;
		color: inherit;
		transition:
			border-color 0.15s ease,
			background-color 0.15s ease;
	}

	.reply-form input::placeholder,
	.reply-form textarea::placeholder {
		opacity: 0.55;
	}

	.reply-form input:focus,
	.reply-form textarea:focus {
		outline: none;
		border-color: var(--primary);
	}

	.reply-form textarea {
		resize: vertical;
		min-height: 4.5rem;
	}

	.form-error,
	.form-success {
		font-size: 0.8rem;
	}

	.form-error {
		color: #dc2626;
	}

	.form-success {
		color: #16a34a;
	}

	.form-actions {
		display: flex;
		justify-content: flex-end;
	}

	.submit-btn {
		gap: 0.375rem;
		padding: 0.5rem 1.25rem;
		border-radius: 0.625rem;
		font-weight: 600;
		font-size: 0.875rem;
	}

	.submit-btn:disabled {
		opacity: 0.6;
		pointer-events: none;
	}

	.empty-state {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		gap: 0.5rem;
		padding: 3.5rem 1rem;
		text-align: center;
	}

	.empty-icon {
		font-size: 3rem;
		opacity: 0.35;
		color: var(--primary);
		margin-bottom: 0.5rem;
	}

	.empty-title {
		font-size: 1rem;
		opacity: 0.75;
	}

	.empty-hint {
		font-size: 0.85rem;
		opacity: 0.55;
		max-width: 32rem;
	}

	.empty-hint code {
		background: var(--inline-code-bg);
		color: var(--inline-code-color);
		padding: 0.1rem 0.3rem;
		border-radius: 0.25rem;
		font-size: 0.8em;
	}

	.retry-btn {
		gap: 0.375rem;
		padding: 0.5rem 1.25rem;
		border-radius: 0.625rem;
		font-weight: 600;
		margin-top: 0.5rem;
	}

	.pagination {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 0.375rem;
		margin: 1.25rem 0 0.25rem;
	}

	.page-btn {
		min-width: 2.25rem;
		height: 2.25rem;
		padding: 0 0.25rem;
		border-radius: 0.5rem;
		display: inline-flex;
		align-items: center;
		justify-content: center;
		background: var(--btn-regular-bg);
		color: var(--btn-content);
		font-size: 0.875rem;
		font-weight: 700;
		transition:
			background-color 0.15s ease,
			transform 0.1s ease,
			opacity 0.15s ease;
	}

	.page-btn:hover:not(:disabled):not(.ellipsis) {
		background: var(--btn-regular-bg-hover);
	}

	.page-btn:active:not(:disabled):not(.ellipsis) {
		transform: scale(0.9);
		background: var(--btn-regular-bg-active);
	}

	.page-btn:disabled {
		opacity: 0.35;
		cursor: not-allowed;
	}

	.page-btn.current {
		background: var(--primary);
		color: #fff;
	}

	:global(html.dark) .page-btn.current {
		color: rgba(0, 0, 0, 0.72);
	}

	.page-btn.ellipsis {
		background: transparent;
		cursor: default;
	}

	.spin {
		display: inline-flex;
		animation: shuoshuo-spin 1s linear infinite;
	}

	@keyframes shuoshuo-spin {
		to {
			transform: rotate(360deg);
		}
	}

	@media (max-width: 640px) {
		.shuoshuo-card {
			padding: 1rem 1.25rem;
		}

		.form-row {
			grid-template-columns: 1fr;
		}

		.latest-reply-head {
			flex-direction: column;
			align-items: flex-start;
			gap: 0.125rem;
		}
	}

	/* 说说正文的 Markdown 样式 */
	:global(.shuoshuo-md) {
		font-size: 0.95rem;
		line-height: 1.7;
		word-break: break-word;
	}

	:global(.shuoshuo-md p) {
		margin: 0.35rem 0;
	}

	:global(.shuoshuo-md p:first-child) {
		margin-top: 0;
	}

	:global(.shuoshuo-md p:last-child) {
		margin-bottom: 0;
	}

	:global(.shuoshuo-md a) {
		color: var(--primary);
		text-decoration: underline;
		text-decoration-style: dashed;
		text-underline-offset: 0.2rem;
		text-decoration-thickness: 1px;
	}

	:global(.shuoshuo-md a:hover) {
		text-decoration-style: solid;
	}

	:global(.shuoshuo-md img) {
		max-width: 100%;
		border-radius: 0.5rem;
	}

	:global(.shuoshuo-md code) {
		background: var(--inline-code-bg);
		color: var(--inline-code-color);
		padding: 0.1rem 0.35rem;
		border-radius: 0.25rem;
		font-size: 0.85em;
	}

	:global(.shuoshuo-md pre) {
		background: var(--codeblock-bg);
		color: #e5e7eb;
		padding: 0.75rem 1rem;
		border-radius: 0.625rem;
		overflow-x: auto;
	}

	:global(.shuoshuo-md pre code) {
		background: transparent;
		color: inherit;
		padding: 0;
	}

	:global(.shuoshuo-md blockquote) {
		border-left: 3px solid var(--primary);
		padding-left: 0.75rem;
		margin: 0.5rem 0;
		opacity: 0.85;
	}

	:global(.shuoshuo-md ul),
	:global(.shuoshuo-md ol) {
		padding-left: 1.5rem;
		margin: 0.35rem 0;
	}

	:global(.shuoshuo-md h1),
	:global(.shuoshuo-md h2),
	:global(.shuoshuo-md h3),
	:global(.shuoshuo-md h4) {
		font-weight: 700;
		margin: 0.75rem 0 0.375rem;
	}

	:global(.shuoshuo-md h1) {
		font-size: 1.25rem;
	}

	:global(.shuoshuo-md h2) {
		font-size: 1.125rem;
	}

	:global(.shuoshuo-md h3),
	:global(.shuoshuo-md h4) {
		font-size: 1rem;
	}

	:global(.shuoshuo-md hr) {
		border: none;
		border-top: 1px dashed var(--line-color);
		margin: 0.75rem 0;
	}

	:global(.shuoshuo-md table) {
		border-collapse: collapse;
		margin: 0.5rem 0;
		display: block;
		overflow-x: auto;
	}

	:global(.shuoshuo-md th),
	:global(.shuoshuo-md td) {
		border: 1px solid var(--line-color);
		padding: 0.375rem 0.625rem;
	}
</style>
