---
{"dg-publish":true,"dg-enable-search":true,"dg-show-tags":true,"dg-permalink":"ai/ai-delegation-framework/","url":"https://notes.kazakov.xyz/ai/ai-delegation-framework/","title":"Фреймворк делегирования задач AI","date":"2026-08-29","status":"published","tags":["ai","ai/agents","ai/delegation","architecture","content/public","public"],"source":"https://arxiv.org/abs/2602.11865","permalink":"/ai/ai-delegation-framework/","dgEnableSearch":true,"dgShowTags":true,"dgPassFrontmatter":true}
---


# Фреймворк делегирования задач AI

Статья Google DeepMind «Intelligent AI Delegation» описывает механику передачи задач агентам: кто получает задачу и на каких правах, как проверяется результат, что происходит при сбое.

<svg viewBox="0 0 900 720" role="img" lang="en" aria-labelledby="archify-diagram-title archify-diagram-description" data-preset="classic" data-quality-profile="showcase" style="" width="900" height="720" xmlns="http://www.w3.org/2000/svg"><style>@font-face { font-family: 'JetBrains Mono'; font-weight: 400; src: local('JetBrains Mono'), local('JetBrainsMono-Regular'); }
@font-face { font-family: 'JetBrains Mono'; font-weight: 500; src: local('JetBrains Mono'), local('JetBrainsMono-Regular'); }
@font-face { font-family: 'JetBrains Mono'; font-weight: 600; src: local('JetBrains Mono'), local('JetBrainsMono-Regular'); }
@font-face { font-family: 'JetBrains Mono'; font-weight: 700; src: local('JetBrains Mono'), local('JetBrainsMono-Regular'); }
svg { font-family: 'JetBrains Mono', ui-monospace, SFMono-Regular, Menlo, Consolas, 'DejaVu Sans Mono', 'Liberation Mono', 'Noto Sans Mono CJK SC', 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', monospace; }
:root, [data-theme="dark"] { --bg: #020617; --grid: #1e293b; --text: #ffffff; --text-muted: #94a3b8; --text-dim: #475569; --text-faint: #7d8da1; --panel: rgba(15, 23, 42, 0.5); --panel-border: #1e293b; --lane-fill: rgba(15, 23, 42, 0.22); --lane-stroke: #334155; --arrow: #64748b; --arrow-emphasis: #34d399; --mask: #0f172a; --frontend-fill: rgba(8, 51, 68, 0.4); --frontend-stroke: #22d3ee; --backend-fill: rgba(6, 78, 59, 0.4); --backend-stroke: #34d399; --database-fill: rgba(76, 29, 149, 0.4); --database-stroke: #a78bfa; --cloud-fill: rgba(120, 53, 15, 0.3); --cloud-stroke: #fbbf24; --security-fill: rgba(136, 19, 55, 0.4); --security-stroke: #fb7185; --messagebus-fill: rgba(251, 146, 60, 0.3); --messagebus-stroke: #fb923c; --external-fill: rgba(30, 41, 59, 0.5); --external-stroke: #94a3b8; --toolbar-bg: rgba(15, 23, 42, 0.8); --toolbar-border: #334155; --toolbar-text: #e2e8f0; --toolbar-hover: rgba(15, 23, 42, 0.95); --toolbar-menu-bg: #0f172a; }
[data-theme="light"] { --bg: #f8fafc; --grid: #e2e8f0; --text: #0f172a; --text-muted: #64748b; --text-dim: #94a3b8; --text-faint: #64748b; --panel: #ffffff; --panel-border: #e2e8f0; --lane-fill: rgba(248, 250, 252, 0.65); --lane-stroke: #cbd5e1; --arrow: #94a3b8; --arrow-emphasis: #059669; --mask: #ffffff; --frontend-fill: rgba(34, 211, 238, 0.15); --frontend-stroke: #0891b2; --backend-fill: rgba(52, 211, 153, 0.18); --backend-stroke: #059669; --database-fill: rgba(167, 139, 250, 0.2); --database-stroke: #7c3aed; --cloud-fill: rgba(251, 191, 36, 0.18); --cloud-stroke: #d97706; --security-fill: rgba(251, 113, 133, 0.15); --security-stroke: #e11d48; --messagebus-fill: rgba(251, 146, 60, 0.15); --messagebus-stroke: #ea580c; --external-fill: rgba(148, 163, 184, 0.18); --external-stroke: #64748b; --toolbar-bg: rgba(255, 255, 255, 0.92); --toolbar-border: #cbd5e1; --toolbar-text: #334155; --toolbar-hover: #ffffff; --toolbar-menu-bg: #ffffff; }
[data-preset="signal-flow"][data-theme="dark"] { --bg: #030711; --grid: #15233a; --panel: rgba(6, 14, 28, 0.78); --panel-border: #1d3350; --lane-fill: rgba(9, 22, 40, 0.5); --lane-stroke: #2c4564; --text: #f5fbff; --text-muted: #9eb0c7; --text-dim: #52667f; --text-faint: #7890ad; --mask: #07101e; --frontend-fill: rgba(6, 182, 212, 0.14); --frontend-stroke: #67e8f9; --backend-fill: rgba(16, 185, 129, 0.14); --backend-stroke: #5eead4; --database-fill: rgba(139, 92, 246, 0.16); --database-stroke: #c4b5fd; --cloud-fill: rgba(245, 158, 11, 0.13); --cloud-stroke: #fcd34d; --security-fill: rgba(244, 63, 94, 0.13); --security-stroke: #fda4af; --messagebus-fill: rgba(249, 115, 22, 0.13); --messagebus-stroke: #fdba74; --external-fill: rgba(71, 85, 105, 0.24); --external-stroke: #a5b4c7; --arrow: #7890ad; --arrow-emphasis: #2dd4bf; --toolbar-bg: rgba(7, 16, 30, 0.86); --toolbar-border: #29415f; --toolbar-menu-bg: #07101e; }
[data-preset="signal-flow"][data-theme="light"] { --bg: #f4f9fc; --grid: #d4e5ee; --panel: rgba(255, 255, 255, 0.82); --panel-border: #bfd5e2; --lane-fill: rgba(232, 243, 248, 0.62); --lane-stroke: #a9c5d5; --text: #102638; --text-muted: #587287; --text-dim: #8aa2b4; --text-faint: #668397; --mask: #ffffff; --frontend-fill: rgba(6, 182, 212, 0.09); --frontend-stroke: #0789a1; --backend-fill: rgba(5, 150, 105, 0.09); --backend-stroke: #087f69; --database-fill: rgba(124, 58, 237, 0.09); --database-stroke: #7254c7; --cloud-fill: rgba(217, 119, 6, 0.08); --cloud-stroke: #b9670b; --security-fill: rgba(225, 29, 72, 0.08); --security-stroke: #c53a59; --messagebus-fill: rgba(234, 88, 12, 0.08); --messagebus-stroke: #c65f27; --external-fill: rgba(100, 116, 139, 0.1); --external-stroke: #607a8c; --arrow: #7b97aa; --arrow-emphasis: #0d9488; }
[data-preset="blueprint"][data-theme="dark"] { --bg: #06131f; --grid: #17425a; --panel: rgba(7, 27, 43, 0.9); --panel-border: #27627f; --lane-fill: rgba(10, 43, 66, 0.36); --lane-stroke: #34799a; --text: #e3f6ff; --text-muted: #91b8ca; --text-dim: #557e92; --text-faint: #739caf; --mask: #0a2031; --frontend-fill: rgba(26, 157, 193, 0.14); --frontend-stroke: #66d9ef; --backend-fill: rgba(31, 155, 124, 0.13); --backend-stroke: #69dfbd; --database-fill: rgba(120, 105, 196, 0.14); --database-stroke: #b4a8ff; --cloud-fill: rgba(197, 145, 43, 0.13); --cloud-stroke: #ffd166; --security-fill: rgba(199, 76, 104, 0.13); --security-stroke: #ff8da1; --messagebus-fill: rgba(207, 112, 49, 0.13); --messagebus-stroke: #ffad66; --external-fill: rgba(84, 119, 139, 0.18); --external-stroke: #a7cad9; --arrow: #78a3b7; --arrow-emphasis: #64dfc1; --toolbar-bg: rgba(7, 28, 45, 0.92); --toolbar-border: #34799a; --toolbar-text: #d8f3ff; --toolbar-hover: rgba(18, 58, 83, 0.95); --toolbar-menu-bg: #0a2031; }
[data-preset="blueprint"][data-theme="light"] { --bg: #edf7fa; --grid: #b5d5e1; --panel: rgba(249, 253, 255, 0.94); --panel-border: #78aabd; --lane-fill: rgba(210, 232, 240, 0.42); --lane-stroke: #83b2c4; --text: #123344; --text-muted: #4e7486; --text-dim: #86a6b4; --text-faint: #668c9d; --mask: #f9fdff; --frontend-fill: rgba(8, 145, 178, 0.08); --frontend-stroke: #087f9c; --backend-fill: rgba(5, 128, 101, 0.08); --backend-stroke: #08755f; --database-fill: rgba(100, 75, 180, 0.08); --database-stroke: #6757a8; --cloud-fill: rgba(181, 120, 15, 0.08); --cloud-stroke: #a86609; --security-fill: rgba(190, 48, 80, 0.07); --security-stroke: #b32f50; --messagebus-fill: rgba(196, 81, 22, 0.07); --messagebus-stroke: #b65120; --external-fill: rgba(79, 112, 128, 0.08); --external-stroke: #506f7e; --arrow: #6d93a5; --arrow-emphasis: #087f69; --toolbar-bg: rgba(247, 252, 254, 0.94); --toolbar-border: #8ab5c5; --toolbar-text: #294f61; --toolbar-hover: #ffffff; --toolbar-menu-bg: #f9fdff; }
[data-preset="editorial"][data-theme="dark"] { --bg: #181611; --grid: #39342a; --panel: rgba(35, 31, 24, 0.96); --panel-border: #625a4a; --lane-fill: rgba(52, 46, 35, 0.46); --lane-stroke: #726957; --text: #f4eddf; --text-muted: #b9ae9b; --text-dim: #776e60; --text-faint: #9d917e; --mask: #231f18; --frontend-fill: rgba(43, 133, 142, 0.16); --frontend-stroke: #7fc6c7; --backend-fill: rgba(63, 132, 92, 0.16); --backend-stroke: #8fc29e; --database-fill: rgba(117, 91, 141, 0.17); --database-stroke: #c0a4d0; --cloud-fill: rgba(170, 119, 49, 0.16); --cloud-stroke: #d8ad68; --security-fill: rgba(157, 69, 65, 0.17); --security-stroke: #df9085; --messagebus-fill: rgba(174, 79, 42, 0.16); --messagebus-stroke: #df946f; --external-fill: rgba(126, 115, 96, 0.18); --external-stroke: #b8ad99; --arrow: #948978; --arrow-emphasis: #dd6b3d; --toolbar-bg: rgba(35, 31, 24, 0.92); --toolbar-border: #625a4a; --toolbar-text: #eee5d5; --toolbar-hover: #302a20; --toolbar-menu-bg: #231f18; }
[data-preset="editorial"][data-theme="light"] { --bg: #f2eee5; --grid: #d8d0c2; --panel: rgba(251, 248, 241, 0.97); --panel-border: #c4b9a6; --lane-fill: rgba(229, 221, 207, 0.42); --lane-stroke: #b8aa94; --text: #242018; --text-muted: #6f6658; --text-dim: #a09788; --text-faint: #817767; --mask: #fbf8f1; --frontend-fill: rgba(31, 117, 126, 0.09); --frontend-stroke: #287e84; --backend-fill: rgba(42, 119, 75, 0.09); --backend-stroke: #397b53; --database-fill: rgba(105, 75, 130, 0.09); --database-stroke: #765d86; --cloud-fill: rgba(157, 103, 31, 0.1); --cloud-stroke: #9a671f; --security-fill: rgba(153, 58, 52, 0.09); --security-stroke: #9e463f; --messagebus-fill: rgba(182, 70, 27, 0.09); --messagebus-stroke: #ad4b25; --external-fill: rgba(101, 91, 75, 0.09); --external-stroke: #746b5e; --arrow: #8a806f; --arrow-emphasis: #bb4c23; --toolbar-bg: rgba(251, 248, 241, 0.94); --toolbar-border: #c4b9a6; --toolbar-text: #40392f; --toolbar-hover: #fffdf8; --toolbar-menu-bg: #fbf8f1; }
svg { width: 100%; min-width: min(900px, 100%); display: block; transform-origin: 0px 0px; transition: transform 0.18s; }
@keyframes archify-lens-in { 
  0% { opacity: 0; transform: translateY(5px) scale(0.985); }
  100% { opacity: 1; transform: translateY(0px) scale(1); }
}
@keyframes archify-guide-in { 
  0% { opacity: 0; transform: translateY(-5px) scale(0.985); }
  100% { opacity: 1; transform: translateY(0px) scale(1); }
}
svg[data-chapter-handoff] [data-node-id][data-chapter-role="stay"] { opacity: 1; }
svg[data-chapter-handoff] [data-node-id][data-chapter-role="enter"] { opacity: 0.78; }
svg[data-chapter-handoff] [data-node-id][data-chapter-role="leave"] { opacity: 0.26; }
@keyframes archify-chapter-anchor { 
  0% { opacity: 0.35; stroke-dashoffset: 8; }
  45% { opacity: 1; }
  100% { opacity: 0.72; stroke-dashoffset: 0; }
}
@keyframes archify-story-caption-in { 
  0% { opacity: 0; transform: translateY(3px); }
  100% { opacity: 1; transform: translateY(0px); }
}
[data-theme="light"] #theme-icon { mask-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 20 20'%3E%3Cg fill='none' stroke='black' stroke-width='1.7' stroke-linecap='round'%3E%3Ccircle cx='10' cy='10' r='3.1'/%3E%3Cpath d='M10 2.2v1.4M10 16.4v1.4M2.2 10h1.4M16.4 10h1.4M4.5 4.5l1 1M14.5 14.5l1 1M15.5 4.5l-1 1M5.5 14.5l-1 1'/%3E%3C/g%3E%3C/svg%3E"); }
[data-theme="dark"] #theme-icon { mask-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 20 20'%3E%3Cpath d='M15.2 12.7A6.3 6.3 0 017.3 4.8a6.3 6.3 0 107.9 7.9Z' fill='none' stroke='black' stroke-width='1.7' stroke-linejoin='round'/%3E%3C/svg%3E"); }
.c-grid { stroke: var(--grid); fill: none; }
.c-mask { fill: var(--mask); stroke: none; }
.c-frontend { fill: var(--frontend-fill); stroke: var(--frontend-stroke); }
.c-backend { fill: var(--backend-fill); stroke: var(--backend-stroke); }
.c-database { fill: var(--database-fill); stroke: var(--database-stroke); }
.c-cloud { fill: var(--cloud-fill); stroke: var(--cloud-stroke); }
.c-security { fill: var(--security-fill); stroke: var(--security-stroke); }
.c-messagebus { fill: var(--messagebus-fill); stroke: var(--messagebus-stroke); }
.c-external { fill: var(--external-fill); stroke: var(--external-stroke); }
.t-primary { fill: var(--text); }
.t-muted { fill: var(--text-muted); }
.t-dim { fill: var(--text-dim); }
.t-frontend { fill: var(--frontend-stroke); }
.t-backend { fill: var(--backend-stroke); }
.t-database { fill: var(--database-stroke); }
.t-cloud { fill: var(--cloud-stroke); }
.t-security { fill: var(--security-stroke); }
.t-messagebus { fill: var(--messagebus-stroke); }
.t-external { fill: var(--external-stroke); }
svg .semantic-sigil { fill: none; stroke: currentcolor; stroke-width: 1.35; stroke-linecap: round; stroke-linejoin: round; opacity: 0.76; pointer-events: none; }
svg .semantic-sigil &gt; * { vector-effect: non-scaling-stroke; }
svg .semantic-sigil .sigil-fill { fill: currentcolor; stroke: none; }
svg .s-frontend { color: var(--frontend-stroke); }
svg .s-backend { color: var(--backend-stroke); }
svg .s-database { color: var(--database-stroke); }
svg .s-cloud { color: var(--cloud-stroke); }
svg .s-security { color: var(--security-stroke); }
svg .s-messagebus { color: var(--messagebus-stroke); }
svg .s-external { color: var(--external-stroke); }
svg .brand-mark { pointer-events: none; }
svg .brand-mark-badge { fill: rgb(255, 255, 255); stroke: none; }
svg .brand-mark-frame { fill: none; stroke: rgb(203, 213, 225); stroke-width: 0.8; vector-effect: non-scaling-stroke; }
svg .brand-mark-fallback { fill: none; stroke: rgb(71, 85, 105); stroke-width: 1.15; stroke-linecap: round; stroke-linejoin: round; }
svg[data-preset="blueprint"] .brand-mark-badge, svg[data-preset="blueprint"] .brand-mark-frame { rx: 1px; }
svg [data-detail] { transition: opacity 160ms; }
svg [data-detail-anchor] { transform-box: fill-box; transform-origin: center center; transition: transform 160ms; }
svg[data-focus-active] [data-focus-match][data-detail], svg[data-focus-active] [data-focus-match] [data-detail], svg[data-reach-active] [data-reach-match][data-detail], svg[data-reach-active] [data-reach-match] [data-detail], svg[data-lens-active] [data-lens-match][data-detail], svg[data-lens-active] [data-lens-match] [data-detail], svg[data-legend-preview-active] [data-legend-preview-match][data-detail], svg[data-legend-preview-active] [data-legend-preview-match] [data-detail], svg[data-intent-trace-active] [data-intent-trace-match][data-detail], svg[data-intent-trace-active] [data-intent-trace-match] [data-detail], svg[data-route-active] [data-route-match][data-detail], svg[data-route-active] [data-route-match] [data-detail], svg[data-story-active] [data-story-step][data-detail], svg[data-story-active] [data-story-step] [data-detail], svg[data-relationship-preview-active] [data-relationship-preview][data-detail], svg[data-relationship-preview-active] [data-relationship-preview] [data-detail], svg [data-node-id]:hover [data-detail], svg [data-node-id]:focus-visible [data-detail] { opacity: 1; pointer-events: auto; }
svg[data-focus-active] [data-focus-match] [data-detail-anchor], svg[data-reach-active] [data-reach-match] [data-detail-anchor], svg[data-lens-active] [data-lens-match] [data-detail-anchor], svg[data-legend-preview-active] [data-legend-preview-match] [data-detail-anchor], svg[data-intent-trace-active] [data-intent-trace-match] [data-detail-anchor], svg[data-route-active] [data-route-match] [data-detail-anchor], svg[data-story-active] [data-story-step] [data-detail-anchor], svg [data-node-id]:hover [data-detail-anchor], svg [data-node-id]:focus-visible [data-detail-anchor] { transform: none; }
.a-default { stroke: var(--arrow); fill: none; }
.a-emphasis { stroke: var(--arrow-emphasis); fill: none; }
.a-security { stroke: var(--security-stroke); fill: none; stroke-dasharray: 5, 5; }
.a-dashed { stroke: var(--database-stroke); fill: none; stroke-dasharray: 4, 4; }
.m-default { fill: var(--arrow); }
.m-emphasis { fill: var(--arrow-emphasis); }
.m-security { fill: var(--security-stroke); }
.m-dashed { fill: var(--database-stroke); }
svg [data-node-id] { cursor: pointer; transition: opacity 0.18s, filter 0.18s; }
svg [data-edge-from] { transition: opacity 0.18s, filter 0.18s; }
svg [data-node-id]:hover, svg [data-node-id]:focus-visible { filter: drop-shadow(0 0 7px var(--frontend-stroke)); outline: none; }
svg [data-node-id]:hover .source-evidence-beacon, svg [data-node-id]:focus-visible .source-evidence-beacon, svg [data-node-id][data-focus-selected] .source-evidence-beacon { opacity: 1; filter: drop-shadow(0 0 4px color-mix(in srgb, var(--backend-stroke) 72%, transparent)); }
svg[data-preset="signal-flow"] .source-evidence-beacon { opacity: 0.9; }
svg[data-preset="blueprint"] .source-evidence-beacon rect { rx: 0.8px; filter: none; }
svg [data-legend-kind][role="button"] { cursor: pointer; }
svg [data-legend-kind]:focus { outline: none; }
svg [data-legend-hit] { fill: transparent; stroke: transparent; stroke-width: 1.4; pointer-events: all; vector-effect: non-scaling-stroke; transition: stroke 150ms, stroke-width 150ms, filter 150ms; }
svg [data-legend-count-badge] { pointer-events: none; }
svg [data-legend-count-badge] rect { fill: var(--mask); stroke: var(--panel-border); stroke-width: 1; vector-effect: non-scaling-stroke; }
svg [data-legend-count-badge] text { fill: var(--text-muted); font-size: 8px; font-weight: 700; text-anchor: middle; }
svg [data-legend-kind][role="button"]:hover [data-legend-hit] { stroke: color-mix(in srgb, var(--database-stroke) 58%, transparent); }
svg [data-legend-kind][role="button"]:focus-visible [data-legend-hit] { stroke: var(--frontend-stroke); stroke-width: 1.8; filter: drop-shadow(0 0 3px var(--frontend-stroke)); }
svg [data-legend-kind][data-legend-selected] [data-legend-hit] { stroke: var(--database-stroke); stroke-width: 2.2; stroke-dasharray: 3, 2; }
svg [data-legend-kind][data-legend-selected] [data-legend-count-badge] rect { stroke: var(--database-stroke); stroke-width: 1.8; }
svg [data-legend-kind][data-legend-zero] { opacity: 0.44; }
svg[data-preset="signal-flow"] [data-legend-kind][data-legend-selected] [data-legend-hit] { filter: drop-shadow(0 0 4px var(--database-stroke)); }
svg[data-preset="blueprint"] [data-legend-hit], svg[data-preset="blueprint"] [data-legend-count-badge] rect { rx: 0; filter: none; }
svg[data-legend-preview-active] [data-node-id], svg[data-legend-preview-active] [data-edge-from] { opacity: 0.11; }
svg[data-legend-preview-active] [data-legend-preview-match] { opacity: 1; }
svg[data-legend-preview-active] [data-legend-preview-peer] { opacity: 0.62; }
svg[data-legend-preview-active] [data-legend-preview-selected] { filter: drop-shadow(0 0 8px var(--database-stroke)); }
svg[data-lens-active] [data-node-id], svg[data-lens-active] [data-edge-from] { opacity: 0.11; }
svg[data-lens-active] [data-lens-match] { opacity: 1; }
svg[data-lens-active] [data-lens-peer] { opacity: 0.62; }
svg[data-lens-active] [data-lens-selected] { filter: drop-shadow(0 0 10px var(--database-stroke)); }
svg[data-preset="signal-flow"] .semantic-lens-flow { stroke-width: 3.7; opacity: 1; }
svg[data-preset="signal-flow"] .semantic-lens-flow[data-direction="out"], svg[data-preset="signal-flow"] .semantic-lens-flow[data-direction="forward"] { filter: drop-shadow(0 0 6px var(--frontend-stroke)); }
svg[data-preset="signal-flow"] .semantic-lens-flow[data-direction="in"], svg[data-preset="signal-flow"] .semantic-lens-flow[data-direction="reverse"] { filter: drop-shadow(0 0 6px var(--database-stroke)); }
svg[data-preset="signal-flow"] .semantic-lens-flow[data-direction="within"] { filter: drop-shadow(0 0 6px var(--messagebus-stroke)); }
svg[data-preset="blueprint"] .semantic-lens-flow { stroke-width: 2.45; stroke-linecap: square; filter: none; opacity: 0.78; }
svg[data-focus-active] [data-node-id], svg[data-focus-active] [data-edge-from] { opacity: 0.13; }
svg[data-focus-active] [data-focus-match] { opacity: 1; }
svg[data-focus-active] [data-focus-selected] { filter: drop-shadow(0 0 10px var(--frontend-stroke)); }
svg[data-reach-active] [data-node-id], svg[data-reach-active] [data-edge-from] { opacity: 0.09; }
svg[data-reach-active] [data-reach-match] { opacity: 1; }
svg[data-reach-active="upstream"] [data-reach-origin] { filter: drop-shadow(0 0 11px var(--database-stroke)); }
svg[data-reach-active="downstream"] [data-reach-origin] { filter: drop-shadow(0 0 11px var(--backend-stroke)); }
svg[data-reach-active="upstream"] [data-edge-from][data-reach-match] { filter: drop-shadow(0 0 4px color-mix(in srgb, var(--database-stroke) 72%, transparent)); }
svg[data-reach-active="downstream"] [data-edge-from][data-reach-match] { filter: drop-shadow(0 0 4px color-mix(in srgb, var(--backend-stroke) 72%, transparent)); }
svg[data-preset="blueprint"][data-reach-active] [data-reach-origin], svg[data-preset="blueprint"][data-reach-active] [data-edge-from][data-reach-match] { filter: none; }
svg[data-preset="blueprint"] .relationship-hit-target:focus-visible .relationship-focus-rail { stroke-linecap: square; filter: none; }
svg[data-relationship-direct-active] [data-node-id], svg[data-relationship-direct-active] [data-edge-from] { opacity: 0.16; }
svg[data-relationship-direct-active] [data-relationship-preview], svg[data-relationship-direct-active] [data-relationship-preview-node] { opacity: 1; }
svg[data-focus-active]:not([data-relationship-pin-active]) .relationship-hit-rail, svg[data-story-active] .relationship-hit-rail, svg[data-route-picking] .relationship-hit-rail, svg[data-route-active] .relationship-hit-rail, svg[data-lens-active] .relationship-hit-rail, svg[data-chapter-preview] .relationship-hit-rail { pointer-events: none; }
svg[data-relationship-preview-active] [data-focus-match] { opacity: 0.18; }
svg[data-relationship-preview-active] [data-relationship-preview], svg[data-relationship-preview-active] [data-relationship-preview-node] { opacity: 1; }
svg[data-relationship-preview-active] [data-relationship-preview] { filter: drop-shadow(0 0 5px var(--arrow-emphasis)); }
svg[data-relationship-preview-active] [data-relationship-preview] path, svg[data-relationship-preview-active] path[data-relationship-preview], svg[data-relationship-preview-active] [data-relationship-preview] line, svg[data-relationship-preview-active] line[data-relationship-preview], svg[data-relationship-preview-active] [data-relationship-preview] polyline, svg[data-relationship-preview-active] polyline[data-relationship-preview] { stroke-width: 2.75; }
svg[data-relationship-preview-active] [data-relationship-preview-source] { filter: drop-shadow(0 0 7px var(--frontend-stroke)); }
svg[data-relationship-preview-active] [data-relationship-preview-target] { filter: drop-shadow(0 0 7px var(--database-stroke)); }
svg[data-preset="signal-flow"] .relationship-flow-pulse { stroke: var(--frontend-stroke); stroke-width: 3.85; filter: drop-shadow(0 0 7px var(--frontend-stroke)); }
svg[data-preset="blueprint"] .relationship-flow-pulse { stroke-width: 2.5; stroke-linecap: square; filter: none; }
svg[data-preset="editorial"] .relationship-flow-pulse { stroke: var(--arrow-emphasis); stroke-width: 2.75; filter: none; }
svg[data-preset="signal-flow"] .semantic-flow-token { filter: drop-shadow(currentcolor 0px 0px 5px); }
svg[data-preset="blueprint"] .semantic-flow-token { filter: none; }
svg[data-preset="editorial"] .semantic-flow-token { filter: none; }
svg[data-intent-trace-active] [data-node-id], svg[data-intent-trace-active] [data-edge-from] { opacity: 0.2; }
svg[data-intent-trace-active] [data-intent-trace-match] { opacity: 1; }
svg[data-intent-trace-active] [data-intent-trace-selected] { filter: drop-shadow(0 0 10px var(--frontend-stroke)); }
svg[data-preset="blueprint"] [data-intent-trace-selected], svg[data-preset="blueprint"] .intent-trace-flow { filter: none; stroke-linecap: square; }
svg[data-preset="editorial"] [data-intent-trace-selected], svg[data-preset="editorial"] .intent-trace-flow { filter: none; }
svg[data-route-picking="target"] [data-node-id] { opacity: 0.24; }
svg[data-route-picking="target"] [data-route-candidate], svg[data-route-picking="target"] [data-route-start] { opacity: 1; }
svg[data-route-picking] [data-node-id] { cursor: crosshair; }
svg[data-route-picking] [data-route-start] { filter: drop-shadow(0 0 10px var(--frontend-stroke)); }
svg[data-route-active] [data-node-id], svg[data-route-active] [data-edge-from] { opacity: 0.11; }
svg[data-route-active] [data-route-match] { opacity: 1; }
svg[data-route-active] [data-route-start] { filter: drop-shadow(0 0 11px var(--frontend-stroke)); }
svg[data-route-active] [data-route-end] { filter: drop-shadow(0 0 11px var(--security-stroke)); }
svg[data-route-active] [data-route-step]:not([data-route-start]):not([data-route-end]) { filter: drop-shadow(0 0 7px var(--backend-stroke)); }
svg[data-route-journey] [data-route-match][data-route-journey-state="past"] { opacity: 0.62; }
svg[data-route-journey] [data-route-match][data-route-journey-state="future"] { opacity: 0.34; }
svg[data-route-journey] [data-route-match][data-route-journey-state="current"] { opacity: 1; }
svg[data-route-journey] [data-node-id][data-route-journey-current] { filter: drop-shadow(0 0 10px var(--backend-stroke)); }
svg[data-route-journey] [data-edge-from][data-route-journey-current] { opacity: 1; filter: drop-shadow(0 0 6px var(--backend-stroke)); }
svg[data-preset="blueprint"] [data-route-start], svg[data-preset="blueprint"] [data-route-end], svg[data-preset="blueprint"] [data-route-step], svg[data-preset="blueprint"] .route-probe-flow { filter: none; stroke-linecap: square; }
svg[data-preset="blueprint"] .route-journey-flow { filter: none; stroke-linecap: square; }
svg[data-preset="editorial"] [data-route-start], svg[data-preset="editorial"] [data-route-end], svg[data-preset="editorial"] [data-route-step], svg[data-preset="editorial"] .route-probe-flow, svg[data-preset="editorial"] .route-journey-flow { filter: none; }
svg[data-preset="editorial"] .c-grid { stroke-dasharray: 1, 4; opacity: 0.48; }
svg[data-preset="editorial"] .c-region { fill: color-mix(in srgb, var(--cloud-fill) 48%, transparent); stroke-dasharray: 9, 4; }
svg[data-preset="editorial"] [data-node-id]:hover, svg[data-preset="editorial"] [data-node-id]:focus-visible, svg[data-preset="editorial"][data-focus-active] [data-focus-selected] { filter: drop-shadow(0 2px 2px color-mix(in srgb, var(--text) 18%, transparent)); }
svg[data-preset="blueprint"] .c-grid { stroke-dasharray: 1, 3; opacity: 0.78; }
svg[data-preset="blueprint"] .c-region { fill: color-mix(in srgb, var(--cloud-fill) 58%, transparent); stroke-dasharray: 12, 4, 2, 4; }
svg[data-preset="blueprint"] .c-security-group, svg[data-preset="blueprint"] .c-lane { stroke-dasharray: 7, 3; }
svg[data-preset="blueprint"] [data-node-id]:hover, svg[data-preset="blueprint"] [data-node-id]:focus-visible { filter: drop-shadow(0 0 3px var(--frontend-stroke)); }
svg[data-preset="blueprint"][data-focus-active] [data-focus-selected] { filter: drop-shadow(0 0 3px var(--frontend-stroke)); }
svg[data-preset="blueprint"][data-focus-active][data-reach-active] [data-focus-selected][data-reach-origin] { filter: none; }
svg[data-preset="blueprint"][data-relationship-preview-active] [data-relationship-preview], svg[data-preset="blueprint"][data-relationship-preview-active] [data-relationship-preview-source], svg[data-preset="blueprint"][data-relationship-preview-active] [data-relationship-preview-target] { filter: drop-shadow(0 0 3px var(--frontend-stroke)); }
svg[data-story-beat] [data-story-step] { opacity: 0.22; filter: saturate(0.42); transition: opacity 180ms, filter 180ms; }
svg[data-story-beat] [data-story-step][data-story-beat-state="past"] { opacity: 0.72; filter: saturate(0.82); }
svg[data-story-beat] [data-story-step][data-story-beat-state="next"] { opacity: 0.5; filter: saturate(0.66); }
svg[data-story-beat] [data-story-step][data-story-beat-state="active"] { opacity: 1; filter: drop-shadow(0 0 7px var(--frontend-stroke)); animation: 360ms cubic-bezier(0.22, 1, 0.36, 1) 0s 1 normal both running archify-story-beat-node; }
svg[data-story-beat] [data-edge-from][data-story-beat-step] { opacity: 0.16; transition: opacity 160ms; }
svg[data-story-beat] [data-edge-from][data-story-beat-step][data-story-beat-state="past"] { opacity: 0.58; }
svg[data-story-beat] [data-edge-from][data-story-beat-step][data-story-beat-state="next"] { opacity: 0.34; }
svg[data-story-beat] [data-edge-from][data-story-beat-step][data-story-beat-state="active"] { opacity: 1; }
svg[data-story-beat] .story-trail-flow { opacity: 0; transition: opacity 160ms; }
svg[data-story-beat] .story-trail-flow[data-story-beat-state="past"] { opacity: 0.34; }
svg[data-story-beat] .story-trail-flow[data-story-beat-state="next"] { opacity: 0.2; }
svg[data-story-beat] .story-trail-flow[data-story-beat-state="active"] { opacity: 0.92; }
svg[data-story-beat] .story-trail-flow[data-story-pulse="true"] { stroke-dasharray: 2, 13; stroke-dashoffset: 30; opacity: 0.92; animation: 0.78s linear 0s 1 normal both running archify-story-flow; }
svg[data-preset="blueprint"] .story-trail-flow { stroke-linecap: square; filter: none; opacity: 0.5; }
svg[data-preset="blueprint"][data-story-beat] [data-story-step][data-story-beat-state="active"] { filter: none; animation: auto ease 0s 1 normal none running none; }
svg[data-preset="editorial"] .story-trail-flow { filter: none; }
svg[data-preset="editorial"][data-story-beat] [data-story-step][data-story-beat-state="active"] { filter: drop-shadow(0 2px 2px color-mix(in srgb, var(--text) 16%, transparent)); }
svg[data-chapter-preview] [data-node-id], svg[data-chapter-preview] [data-edge-from] { opacity: 0.1 !important; filter: none !important; transition: none !important; }
svg[data-chapter-preview] .story-trail-overlay { opacity: 0.08 !important; }
svg[data-chapter-preview] [data-node-id][data-chapter-preview-role="stay"] { opacity: 0.76 !important; filter: saturate(0.7) !important; }
svg[data-chapter-preview] [data-node-id][data-chapter-preview-role="enter"] { opacity: 1 !important; filter: saturate(1.2) !important; }
svg[data-chapter-preview] [data-node-id][data-chapter-preview-role="leave"] { opacity: 0.28 !important; filter: grayscale(0.72) !important; }
svg[data-preset="blueprint"][data-chapter-preview] [data-node-id] { filter: none !important; }
.c-security-group { fill: transparent; stroke: var(--security-stroke); stroke-dasharray: 4, 4; }
.c-region { fill: rgba(251, 191, 36, 0.05); stroke: var(--cloud-stroke); stroke-dasharray: 8, 4; }
.c-lane { fill: var(--lane-fill); stroke: var(--lane-stroke); stroke-dasharray: 6, 6; }
svg[data-preset="signal-flow"][data-animation="trace"] [data-animate="edge"] { stroke-linecap: round; filter: drop-shadow(0 0 3px var(--arrow-emphasis)); animation-duration: 1.75s; }
svg[data-preset="signal-flow"][data-animation="trace"] [data-animate="node"] { animation-duration: 3.1s; }
svg[data-preset="blueprint"][data-animation="trace"] [data-animate="edge"] { stroke-linecap: square; filter: none; animation-duration: 2.15s; }
svg[data-preset="blueprint"][data-animation="trace"] [data-animate="node"] { animation-name: archify-blueprint-node-pulse; animation-duration: 3.8s; }
svg[data-preset="editorial"][data-animation="trace"] [data-animate="edge"] { filter: none; animation-duration: 2.2s; }
svg[data-preset="editorial"][data-animation="trace"] [data-animate="node"] { animation-name: archify-editorial-node-pulse; animation-duration: 3.5s; }
@keyframes archify-edge-flow { 
  0% { stroke-dasharray: 10, 8; stroke-dashoffset: 54; opacity: 0.42; }
  88% { stroke-dasharray: 10, 8; stroke-dashoffset: 0; opacity: 1; }
  99.9% { stroke-dasharray: 10, 8; stroke-dashoffset: 0; opacity: 1; }
  100% { stroke-dashoffset: 0; opacity: 1; }
}
@keyframes archify-node-pulse { 
  0%, 72%, 100% { filter: none; stroke-width: 1.5; }
  18%, 36% { filter: drop-shadow(0 0 8px var(--arrow-emphasis)); stroke-width: 2.4; }
}
@keyframes archify-blueprint-node-pulse { 
  0%, 72%, 100% { opacity: 1; filter: none; }
  18%, 36% { opacity: 0.72; filter: drop-shadow(0 0 2px var(--frontend-stroke)); }
}
@keyframes archify-editorial-node-pulse { 
  0%, 72%, 100% { opacity: 1; filter: none; }
  18%, 36% { opacity: 0.78; filter: drop-shadow(0 2px 2px color-mix(in srgb, var(--text) 18%, transparent)); }
}
@keyframes archify-signal-scan { 
  0%, 18% { opacity: 1; transform: translateX(-70%); }
  70% { opacity: 1; transform: translateX(70%); }
  100% { opacity: 0; transform: translateX(70%); }
}
@keyframes archify-radar-live { 
  0% { box-shadow: 0 0 0 0 color-mix(in srgb, var(--backend-stroke) 52%, transparent); }
  55%, 100% { box-shadow: transparent 0px 0px 0px 0.32rem; }
}
@keyframes archify-guided-progress { 
  0% { transform: scaleX(var(--guided-progress-start, 0)); }
  100% { transform: scaleX(1); }
}
@keyframes archify-story-flow { 
  100% { stroke-dashoffset: 0; }
}
@keyframes archify-intent-trace-flow { 
  0% { opacity: 0.28; stroke-dashoffset: 0; }
  12%, 78% { opacity: 0.94; }
  100% { opacity: 0; stroke-dashoffset: -1; }
}
@keyframes archify-route-probe-flow { 
  0% { stroke-dashoffset: 0; opacity: 0.96; }
  78% { stroke-dashoffset: -1; opacity: 0.96; }
  100% { stroke-dasharray: none; stroke-dashoffset: 0; opacity: 0.58; }
}
@keyframes archify-route-journey-flow { 
  0% { opacity: 0.16; stroke-dashoffset: 0; }
  14%, 76% { opacity: 1; }
  100% { opacity: 0; stroke-dashoffset: -1; }
}
@keyframes archify-semantic-lens-flow { 
  0% { opacity: 0.2; stroke-dashoffset: 0; }
  12%, 78% { opacity: 0.9; }
  100% { opacity: 0; stroke-dashoffset: -1; }
}
@keyframes archify-relationship-pulse { 
  0% { opacity: 0.18; stroke-dashoffset: 0; }
  10%, 76% { opacity: 0.98; }
  100% { opacity: 0; stroke-dashoffset: -1; }
}
@keyframes archify-relationship-token-life { 
  0%, 7% { opacity: 0; }
  13%, 82% { opacity: 1; }
  100% { opacity: 0; }
}
@keyframes archify-story-beat-node { 
  0% { opacity: 0.48; filter: saturate(0.6); }
  58% { opacity: 1; filter: drop-shadow(0 0 10px var(--frontend-stroke)); }
  100% { opacity: 1; filter: drop-shadow(0 0 7px var(--frontend-stroke)); }
}
@keyframes archify-share-cue-enter { 
  0% { opacity: 0; transform: translate(-50%, -0.45rem); }
  100% { opacity: 1; transform: translate(-50%, 0px); }
}
@keyframes archify-share-cue-pulse { 
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.48; transform: scale(0.78); }
}
:root, svg { --bg: #020617; --grid: #1e293b; --text: #ffffff; --text-muted: #94a3b8; --text-dim: #475569; --text-faint: #7d8da1; --panel: rgba(15, 23, 42, 0.5); --panel-border: #1e293b; --lane-fill: rgba(15, 23, 42, 0.22); --lane-stroke: #334155; --arrow: #64748b; --arrow-emphasis: #34d399; --mask: #0f172a; --frontend-fill: rgba(8, 51, 68, 0.4); --frontend-stroke: #22d3ee; --backend-fill: rgba(6, 78, 59, 0.4); --backend-stroke: #34d399; --database-fill: rgba(76, 29, 149, 0.4); --database-stroke: #a78bfa; --cloud-fill: rgba(120, 53, 15, 0.3); --cloud-stroke: #fbbf24; --security-fill: rgba(136, 19, 55, 0.4); --security-stroke: #fb7185; --messagebus-fill: rgba(251, 146, 60, 0.3); --messagebus-stroke: #fb923c; --external-fill: rgba(30, 41, 59, 0.5); --external-stroke: #94a3b8; --toolbar-bg: rgba(15, 23, 42, 0.8); --toolbar-border: #334155; --toolbar-text: #e2e8f0; --toolbar-hover: rgba(15, 23, 42, 0.95); --toolbar-menu-bg: #0f172a; }
@media (prefers-color-scheme: light) { :root, svg { --bg: #f8fafc; --grid: #e2e8f0; --text: #0f172a; --text-muted: #64748b; --text-dim: #94a3b8; --text-faint: #64748b; --panel: #ffffff; --panel-border: #e2e8f0; --lane-fill: rgba(248, 250, 252, 0.65); --lane-stroke: #cbd5e1; --arrow: #94a3b8; --arrow-emphasis: #059669; --mask: #ffffff; --frontend-fill: rgba(34, 211, 238, 0.15); --frontend-stroke: #0891b2; --backend-fill: rgba(52, 211, 153, 0.18); --backend-stroke: #059669; --database-fill: rgba(167, 139, 250, 0.2); --database-stroke: #7c3aed; --cloud-fill: rgba(251, 191, 36, 0.18); --cloud-stroke: #d97706; --security-fill: rgba(251, 113, 133, 0.15); --security-stroke: #e11d48; --messagebus-fill: rgba(251, 146, 60, 0.15); --messagebus-stroke: #ea580c; --external-fill: rgba(148, 163, 184, 0.18); --external-stroke: #64748b; --toolbar-bg: rgba(255, 255, 255, 0.92); --toolbar-border: #cbd5e1; --toolbar-text: #334155; --toolbar-hover: #ffffff; --toolbar-menu-bg: #ffffff; } }
svg[data-theme="light"] { --bg: #f8fafc; --grid: #e2e8f0; --text: #0f172a; --text-muted: #64748b; --text-dim: #94a3b8; --text-faint: #64748b; --panel: #ffffff; --panel-border: #e2e8f0; --lane-fill: rgba(248, 250, 252, 0.65); --lane-stroke: #cbd5e1; --arrow: #94a3b8; --arrow-emphasis: #059669; --mask: #ffffff; --frontend-fill: rgba(34, 211, 238, 0.15); --frontend-stroke: #0891b2; --backend-fill: rgba(52, 211, 153, 0.18); --backend-stroke: #059669; --database-fill: rgba(167, 139, 250, 0.2); --database-stroke: #7c3aed; --cloud-fill: rgba(251, 191, 36, 0.18); --cloud-stroke: #d97706; --security-fill: rgba(251, 113, 133, 0.15); --security-stroke: #e11d48; --messagebus-fill: rgba(251, 146, 60, 0.15); --messagebus-stroke: #ea580c; --external-fill: rgba(148, 163, 184, 0.18); --external-stroke: #64748b; --toolbar-bg: rgba(255, 255, 255, 0.92); --toolbar-border: #cbd5e1; --toolbar-text: #334155; --toolbar-hover: #ffffff; --toolbar-menu-bg: #ffffff; }
svg[data-theme="dark"] { --bg: #020617; --grid: #1e293b; --text: #ffffff; --text-muted: #94a3b8; --text-dim: #475569; --text-faint: #7d8da1; --panel: rgba(15, 23, 42, 0.5); --panel-border: #1e293b; --lane-fill: rgba(15, 23, 42, 0.22); --lane-stroke: #334155; --arrow: #64748b; --arrow-emphasis: #34d399; --mask: #0f172a; --frontend-fill: rgba(8, 51, 68, 0.4); --frontend-stroke: #22d3ee; --backend-fill: rgba(6, 78, 59, 0.4); --backend-stroke: #34d399; --database-fill: rgba(76, 29, 149, 0.4); --database-stroke: #a78bfa; --cloud-fill: rgba(120, 53, 15, 0.3); --cloud-stroke: #fbbf24; --security-fill: rgba(136, 19, 55, 0.4); --security-stroke: #fb7185; --messagebus-fill: rgba(251, 146, 60, 0.3); --messagebus-stroke: #fb923c; --external-fill: rgba(30, 41, 59, 0.5); --external-stroke: #94a3b8; --toolbar-bg: rgba(15, 23, 42, 0.8); --toolbar-border: #334155; --toolbar-text: #e2e8f0; --toolbar-hover: rgba(15, 23, 42, 0.95); --toolbar-menu-bg: #0f172a; }
rect.c-bg-rect { fill: var(--bg); }
</style><rect width="100%" height="100%" class="c-bg-rect"/>
        <title id="archify-diagram-title">Делегирование задачи AI-агенту</title>
        <desc id="archify-diagram-description">A workflow diagram generated by Archify.</desc>
        <!-- Definitions -->
        <defs>
          <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
            <polygon points="0 0, 10 3.5, 0 7" class="m-default"/>
          </marker>
          <marker id="arrowhead-emphasis" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
            <polygon points="0 0, 10 3.5, 0 7" class="m-emphasis"/>
          </marker>
          <marker id="arrowhead-security" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
            <polygon points="0 0, 10 3.5, 0 7" class="m-security"/>
          </marker>
          <marker id="arrowhead-dashed" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
            <polygon points="0 0, 10 3.5, 0 7" class="m-dashed"/>
          </marker>
          <pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
            <path d="M 40 0 L 0 0 0 40" class="c-grid" stroke-width="0.5"/>
          </pattern>
        </defs>

        <!-- Background Grid -->
        <rect width="100%" height="100%" fill="url(#grid)"/>

        <!-- Swimlanes -->
        <rect data-graph-role="structural-frame" data-composition-frame-kind="lane" data-composition-frame-id="lane-0" x="40" y="52" width="640" height="104" rx="10" class="c-lane" stroke-width="1"/>
        <text x="54" y="74" class="t-dim" font-size="10" font-weight="600">01 / Цепочка решений</text>

        <rect data-graph-role="structural-frame" data-composition-frame-kind="lane" data-composition-frame-id="lane-1" x="40" y="176" width="640" height="104" rx="10" class="c-lane" stroke-width="1"/>
        <text x="54" y="198" class="t-dim" font-size="10" font-weight="600">02 / Права доступа</text>

        <rect data-graph-role="structural-frame" data-composition-frame-kind="lane" data-composition-frame-id="lane-2" x="40" y="300" width="640" height="104" rx="10" class="c-lane" stroke-width="1"/>
        <text x="54" y="322" class="t-dim" font-size="10" font-weight="600">03 / Исполнение и проверка</text>

        <rect data-graph-role="structural-frame" data-composition-frame-kind="lane" data-composition-frame-id="lane-3" x="40" y="424" width="640" height="104" rx="10" class="c-lane" stroke-width="1"/>
        <rect data-graph-role="structural-frame" data-composition-frame-kind="exception-lane" data-composition-frame-id="lane-3-exception" x="46" y="430" width="628" height="92" rx="8" class="c-security-group" stroke-width="1"/>
        <text x="54" y="446" class="t-security" font-size="10" font-weight="600">EX / Сбой</text>

        <rect data-graph-role="structural-frame" data-composition-frame-kind="lane" data-composition-frame-id="lane-4" x="40" y="548" width="640" height="104" rx="10" class="c-lane" stroke-width="1"/>
        <text x="54" y="570" class="t-dim" font-size="10" font-weight="600">05 / Журнал</text>

        <!-- Phase headers -->
        <line x1="42" y1="35" x2="266" y2="35" class="a-default" stroke-width="1.1"/>
        <rect x="42" y="27" width="224" height="16" rx="4" class="c-mask"/>
        <text x="154" y="39" class="t-muted" font-size="8" font-weight="600" text-anchor="middle">Решить</text>
        <line x1="254" y1="35" x2="476" y2="35" class="a-emphasis" stroke-width="1.1"/>
        <rect x="254" y="27" width="222" height="16" rx="4" class="c-mask"/>
        <text x="365" y="39" class="t-backend" font-size="8" font-weight="600" text-anchor="middle">Дать права</text>
        <line x1="454" y1="35" x2="671" y2="35" class="a-dashed" stroke-width="1.1"/>
        <rect x="454" y="27" width="217" height="16" rx="4" class="c-mask"/>
        <text x="562.5" y="39" class="t-messagebus" font-size="8" font-weight="600" text-anchor="middle">Исполнить и проверить</text>

        <!-- Workflow groups -->


        <!-- Edge paths -->
        <path data-edge-from="task" data-edge-to="contract" data-edge-key="0" data-edge-id="e-task-contract" data-composition-points="136,119;172,119" d="M 136 119 L 172 119" class="a-default" stroke-width="1.4" marker-end="url(#arrowhead)"/>
        <path data-edge-from="contract" data-edge-to="trust" data-edge-key="1" data-edge-id="e-contract-trust" data-composition-points="220,145;220,243;252,243" d="M 220 145 L 220 243 L 252 243" class="a-default" stroke-width="1.4" marker-end="url(#arrowhead)"/>
        <path data-edge-from="trust" data-edge-to="level" data-edge-key="2" data-edge-id="e-trust-level" data-composition-points="348,243;382,243" d="M 348 243 L 382 243" class="a-default" stroke-width="1.4" marker-end="url(#arrowhead)"/>
        <path data-edge-from="level" data-edge-to="run" data-edge-key="3" data-edge-id="e-level-run" data-composition-points="430,269;430,367;452,367" d="M 430 269 L 430 367 L 452 367" class="a-emphasis" stroke-width="1.8" marker-end="url(#arrowhead-emphasis)"/>
        <path data-edge-from="run" data-edge-to="validate" data-edge-key="4" data-edge-id="e-run-validate" data-composition-points="548,367;577,367" d="M 548 367 L 577 367" class="a-default" stroke-width="1.4" marker-end="url(#arrowhead)"/>
        <path data-edge-from="validate" data-edge-to="done" data-edge-label="ок" data-edge-key="5" data-edge-id="e-validate-done" data-composition-points="625,341;625,228;625,228;625,145" d="M 625 341 L 625 228 L 625 228 L 625 145" class="a-emphasis" stroke-width="1.8" marker-end="url(#arrowhead-emphasis)"/>
        <path data-edge-from="validate" data-edge-to="fallback" data-edge-label="не прошла" data-edge-key="6" data-edge-id="e-validate-fallback" data-composition-points="625,393;625,491;548,491" d="M 625 393 L 625 491 L 548 491" class="a-security" stroke-width="1.4" marker-end="url(#arrowhead-security)"/>
        <path data-edge-from="fallback" data-edge-to="trust" data-edge-label="понизить уровень" data-edge-key="7" data-edge-id="e-fallback-trust" data-composition-points="452,491;300,491;300,277" d="M 452 491 L 300 491 L 300 277" class="a-dashed" stroke-width="1.4" marker-end="url(#arrowhead-dashed)"/>
        <path data-edge-from="validate" data-edge-to="log" data-edge-label="итог" data-edge-key="8" data-edge-id="e-run-log" data-composition-points="673,367;692,367;692,615;673,615" d="M 673 367 L 692 367 L 692 615 L 673 615" class="a-dashed" stroke-width="1.4" marker-end="url(#arrowhead-dashed)"/>

        <!-- Nodes -->
        <g id="node-task" data-node-id="task" data-node-label="Задача" tabindex="0" role="button" aria-label="Focus Задача, делегировать или нет?, Цепочка решений › Решить" aria-pressed="false" data-node-kind="external" data-node-sublabel="делегировать или нет?" data-node-context="Цепочка решений › Решить">
          <title>Задача · делегировать или нет? · Цепочка решений › Решить</title>
          <rect x="40" y="93" width="96" height="52" rx="6" class="c-mask"/>
          <rect x="40" y="93" width="96" height="52" rx="6" class="c-external" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="external" class="semantic-sigil s-external" transform="translate(46 99) scale(0.6875)">
            <rect x="2.5" y="5" width="8.5" height="8" rx="1.5"/>
            <path d="M8 2.5h5.5V8M13.5 2.5 7.5 8.5"/>
          </g>
          <text data-node-label="" x="88" y="114" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Задача</text>
          <text x="88" y="131" class="t-muted" font-size="6.9" text-anchor="middle">делегировать или нет?</text>
        </g>

        <g id="node-done" data-node-id="done" data-node-label="Принято" tabindex="0" role="button" aria-label="Focus Принято, с доказательством, Цепочка решений › Исполнить и проверить" aria-pressed="false" data-node-kind="backend" data-node-sublabel="с доказательством" data-node-context="Цепочка решений › Исполнить и проверить">
          <title>Принято · с доказательством · Цепочка решений › Исполнить и проверить</title>
          <rect x="577" y="93" width="96" height="52" rx="6" class="c-mask"/>
          <rect x="577" y="93" width="96" height="52" rx="6" class="c-backend" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="backend" class="semantic-sigil s-backend" transform="translate(583 99) scale(0.6875)">
            <path d="M6 3 3 8l3 5M10 3l3 5-3 5"/>
          </g>
          <text data-node-label="" x="625" y="114" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Принято</text>
          <text x="625" y="131" class="t-muted" font-size="8" text-anchor="middle">с доказательством</text>
        </g>

        <g id="node-contract" data-node-id="contract" data-node-label="Контракт" tabindex="0" role="button" aria-label="Focus Контракт, что считается успехом, Цепочка решений › Решить" aria-pressed="false" data-node-kind="backend" data-node-sublabel="что считается успехом" data-node-context="Цепочка решений › Решить">
          <title>Контракт · что считается успехом · Цепочка решений › Решить</title>
          <rect x="172" y="93" width="96" height="52" rx="6" class="c-mask"/>
          <rect x="172" y="93" width="96" height="52" rx="6" class="c-backend" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="backend" class="semantic-sigil s-backend" transform="translate(178 99) scale(0.6875)">
            <path d="M6 3 3 8l3 5M10 3l3 5-3 5"/>
          </g>
          <text data-node-label="" x="220" y="114" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Контракт</text>
          <text x="220" y="131" class="t-muted" font-size="6.9" text-anchor="middle">что считается успехом</text>
        </g>

        <g id="node-trust" data-node-id="trust" data-node-label="Готовность" tabindex="0" role="button" aria-label="Focus Готовность, история, цена ошибки, Права доступа › Дать права" aria-pressed="false" data-node-kind="security" data-node-sublabel="история, цена ошибки" data-node-tag="пересчёт каждый раз" data-node-context="Права доступа › Дать права">
          <title>Готовность · история, цена ошибки · Права доступа › Дать права · пересчёт каждый раз</title>
          <rect x="252" y="209" width="96" height="68" rx="6" class="c-mask"/>
          <rect x="252" y="209" width="96" height="68" rx="6" class="c-security" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="security" class="semantic-sigil s-security" transform="translate(258 215) scale(0.6875)">
            <path d="M8 2.2 13 4v3.5c0 3.1-1.8 5.4-5 6.5-3.2-1.1-5-3.4-5-6.5V4Z"/>
            <path d="m5.8 8 1.5 1.5 3-3"/>
          </g>
          <text data-node-label="" x="300" y="230" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Готовность</text>
          <text x="300" y="247" class="t-muted" font-size="7.3" text-anchor="middle">история, цена ошибки</text>
        <text x="300" y="265" class="t-security" font-size="7" text-anchor="middle">пересчёт каждый раз</text>
        </g>

        <g id="node-level" data-node-id="level" data-node-label="Уровень 0–4" tabindex="0" role="button" aria-label="Focus Уровень 0–4, от чтения до критических, Права доступа › Дать права" aria-pressed="false" data-node-kind="security" data-node-sublabel="от чтения до критических" data-node-context="Права доступа › Дать права">
          <title>Уровень 0–4 · от чтения до критических · Права доступа › Дать права</title>
          <rect x="382" y="217" width="96" height="52" rx="6" class="c-mask"/>
          <rect x="382" y="217" width="96" height="52" rx="6" class="c-security" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="security" class="semantic-sigil s-security" transform="translate(388 223) scale(0.6875)">
            <path d="M8 2.2 13 4v3.5c0 3.1-1.8 5.4-5 6.5-3.2-1.1-5-3.4-5-6.5V4Z"/>
            <path d="m5.8 8 1.5 1.5 3-3"/>
          </g>
          <text data-node-label="" x="430" y="238" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Уровень 0–4</text>
          <text x="430" y="255" class="t-muted" font-size="6.1" text-anchor="middle">от чтения до критических</text>
        </g>

        <g id="node-run" data-node-id="run" data-node-label="Исполнение" tabindex="0" role="button" aria-label="Focus Исполнение, чекпоинт, timeout, Исполнение и проверка › Исполнить и проверить" aria-pressed="false" data-node-kind="messagebus" data-node-sublabel="чекпоинт, timeout" data-node-context="Исполнение и проверка › Исполнить и проверить">
          <title>Исполнение · чекпоинт, timeout · Исполнение и проверка › Исполнить и проверить</title>
          <rect x="452" y="341" width="96" height="52" rx="6" class="c-mask"/>
          <rect x="452" y="341" width="96" height="52" rx="6" class="c-messagebus" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="messagebus" class="semantic-sigil s-messagebus" transform="translate(458 347) scale(0.6875)">
            <path d="M2.5 4.5h11M2.5 8h11M2.5 11.5h11"/>
            <circle cx="5" cy="4.5" r="1" class="sigil-fill"/>
            <circle cx="10.5" cy="8" r="1" class="sigil-fill"/>
            <circle cx="7" cy="11.5" r="1" class="sigil-fill"/>
          </g>
          <text data-node-label="" x="500" y="362" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Исполнение</text>
          <text x="500" y="379" class="t-muted" font-size="8" text-anchor="middle">чекпоинт, timeout</text>
        </g>

        <g id="node-validate" data-node-id="validate" data-node-label="Валидация" tabindex="0" role="button" aria-label="Focus Валидация, сверка с контрактом, Исполнение и проверка › Исполнить и проверить" aria-pressed="false" data-node-kind="security" data-node-sublabel="сверка с контрактом" data-node-context="Исполнение и проверка › Исполнить и проверить">
          <title>Валидация · сверка с контрактом · Исполнение и проверка › Исполнить и проверить</title>
          <rect x="577" y="341" width="96" height="52" rx="6" class="c-mask"/>
          <rect x="577" y="341" width="96" height="52" rx="6" class="c-security" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="security" class="semantic-sigil s-security" transform="translate(583 347) scale(0.6875)">
            <path d="M8 2.2 13 4v3.5c0 3.1-1.8 5.4-5 6.5-3.2-1.1-5-3.4-5-6.5V4Z"/>
            <path d="m5.8 8 1.5 1.5 3-3"/>
          </g>
          <text data-node-label="" x="625" y="362" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Валидация</text>
          <text x="625" y="379" class="t-muted" font-size="7.7" text-anchor="middle">сверка с контрактом</text>
        </g>

        <g id="node-fallback" data-node-id="fallback" data-node-label="Fallback" tabindex="0" role="button" aria-label="Focus Fallback, откат / другой агент, Сбой › Исполнить и проверить" aria-pressed="false" data-node-kind="security" data-node-sublabel="откат / другой агент" data-node-context="Сбой › Исполнить и проверить">
          <title>Fallback · откат / другой агент · Сбой › Исполнить и проверить</title>
          <rect x="452" y="465" width="96" height="52" rx="6" class="c-mask"/>
          <rect x="452" y="465" width="96" height="52" rx="6" class="c-security" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="security" class="semantic-sigil s-security" transform="translate(458 471) scale(0.6875)">
            <path d="M8 2.2 13 4v3.5c0 3.1-1.8 5.4-5 6.5-3.2-1.1-5-3.4-5-6.5V4Z"/>
            <path d="m5.8 8 1.5 1.5 3-3"/>
          </g>
          <text data-node-label="" x="500" y="486" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Fallback</text>
          <text x="500" y="503" class="t-muted" font-size="7.3" text-anchor="middle">откат / другой агент</text>
        </g>

        <g id="node-log" data-node-id="log" data-node-label="Журнал" tabindex="0" role="button" aria-label="Focus Журнал, кто, кому, права, итог, Журнал › Исполнить и проверить" aria-pressed="false" data-node-kind="database" data-node-sublabel="кто, кому, права, итог" data-node-context="Журнал › Исполнить и проверить">
          <title>Журнал · кто, кому, права, итог · Журнал › Исполнить и проверить</title>
          <rect x="577" y="589" width="96" height="52" rx="6" class="c-mask"/>
          <rect x="577" y="589" width="96" height="52" rx="6" class="c-database" stroke-width="1.5"/>
          <g aria-hidden="true" data-semantic-sigil="database" class="semantic-sigil s-database" transform="translate(583 595) scale(0.6875)">
            <ellipse cx="8" cy="4" rx="5" ry="2"/>
            <path d="M3 4v8c0 1.1 2.2 2 5 2s5-.9 5-2V4M3 8c0 1.1 2.2 2 5 2s5-.9 5-2"/>
          </g>
          <text data-node-label="" x="625" y="610" class="t-primary" font-size="11" font-weight="600" text-anchor="middle">Журнал</text>
          <text x="625" y="627" class="t-muted" font-size="6.6" text-anchor="middle">кто, кому, права, итог</text>
        </g>

        <!-- Edge labels -->





        <g data-edge-from="validate" data-edge-to="done" data-edge-label="ок" data-edge-key="5" data-edge-id="e-validate-done">
          <rect x="610" y="208" width="30" height="14" rx="3" class="c-mask"/>
          <text x="625" y="218" class="t-backend" font-size="8" text-anchor="middle">ок</text>
        </g>
        <g data-edge-from="validate" data-edge-to="fallback" data-edge-label="не прошла" data-edge-key="6" data-edge-id="e-validate-fallback">
          <rect x="598.4" y="432" width="53.199999999999996" height="14" rx="3" class="c-mask"/>
          <text x="625" y="442" class="t-security" font-size="8" text-anchor="middle">не прошла</text>
        </g>
        <g data-edge-from="fallback" data-edge-to="trust" data-edge-label="понизить уровень" data-edge-key="7" data-edge-id="e-fallback-trust">
          <rect x="256.6" y="374" width="86.8" height="14" rx="3" class="c-mask"/>
          <text x="300" y="384" class="t-database" font-size="8" text-anchor="middle">понизить уровень</text>
        </g>
        <g data-edge-from="validate" data-edge-to="log" data-edge-label="итог" data-edge-key="8" data-edge-id="e-run-log">
          <rect x="677" y="471" width="30" height="14" rx="3" class="c-mask"/>
          <text x="692" y="481" class="t-database" font-size="8" text-anchor="middle">итог</text>
        </g>

        <!-- Legend -->
        <g data-legend="">
          <text x="20" y="676" class="t-primary" font-size="12" font-weight="650">Legend</text>
          <g data-legend-semantic-kind="backend" data-legend-x="20" data-legend-baseline="696" data-legend-width="91">
            <rect x="20" y="688" width="14" height="9" rx="2" class="c-backend" stroke-width="1"/>
            <text x="42" y="696" class="t-muted" font-size="7.5" font-weight="500">Agent logic</text>
          </g>
          <g data-legend-semantic-kind="security" data-legend-x="118" data-legend-baseline="696" data-legend-width="70">
            <rect x="118" y="688" width="14" height="9" rx="2" class="c-security" stroke-width="1"/>
            <text x="140" y="696" class="t-muted" font-size="7.5" font-weight="500">Policy</text>
          </g>
          <g data-legend-semantic-kind="messagebus" data-legend-x="195" data-legend-baseline="696" data-legend-width="91">
            <rect x="195" y="688" width="14" height="9" rx="2" class="c-messagebus" stroke-width="1"/>
            <text x="217" y="696" class="t-muted" font-size="7.5" font-weight="500">Tool action</text>
          </g>
          <g data-legend-semantic-kind="database" data-legend-x="293" data-legend-baseline="696" data-legend-width="109">
            <rect x="293" y="688" width="14" height="9" rx="2" class="c-database" stroke-width="1"/>
            <text x="315" y="696" class="t-muted" font-size="7.5" font-weight="500">Context / trace</text>
          </g>
          <g data-legend-semantic-kind="external" data-legend-x="409" data-legend-baseline="696" data-legend-width="109">
            <rect x="409" y="688" width="14" height="9" rx="2" class="c-external" stroke-width="1"/>
            <text x="431" y="696" class="t-muted" font-size="7.5" font-weight="500">External system</text>
          </g>
        </g>
      </svg>

Схема целиком: контракт задаёт критерий успеха, права выдаются по уровню готовности агента, результат проверяется против контракта, при сбое включается fallback и уровень доступа понижается. Всё пишется в журнал.

## Проблема: жёсткие системы ломаются

Большинство систем сейчас работает на if-then правилах. Пока сценарий предусмотрен, они предсказуемы. Первый непредусмотренный случай — и система либо падает, либо делает что-то опасное.

Пример: микросервис A ждёт ответа от B тридцать секунд, B сегодня медленный, правила на этот случай нет. Timeout, дальше не работает ничего.

В статье жёсткие правила предлагается заменить рынком, где агенты договариваются о задачах через контракты. Система сама решает, кому отдать задачу, на каких условиях и что делать при сбое.

## Цепочка решений при делегировании

Отдавая задачу агенту, отвечаешь на четыре вопроса подряд:

1. Нужно ли вообще передавать. Задача может быть тривиальной или слишком рискованной.
2. Как сформулировать задачу, чтобы агент понял, что здесь считается успехом.
3. Какие права дать: потратить $100, записать миллион строк в БД или только читать.
4. Как проверить результат. На веру не берём, нужна механика валидации.

Ответы не фиксируются один раз. По ходу работы условия меняются: права пересматриваются, сложность растёт, появляется новая информация. Система должна уметь передать задачу другому агенту, откатиться на предыдущий этап или включить план B.

## Доверие: over-delegating и under-delegating

Over-delegating — задачу отдают агенту, который к ней не готов. Ошибка в проде, потеря денег, утечка данных.

Under-delegating — всё делается руками, хотя агент справился бы быстрее и точнее. Потерянное время и сдвинутые сроки.

Фреймворк взвешивает сложность задачи, историческую точность агента на похожих задачах и стоимость ошибки. На этом решает: агент готов или нужна подстраховка.

## Безопасность и прозрачность

Как доверять агенту, если не видно, что он делает? Вместо рейтингов вида ★★★★☆ в статье предлагаются проверяемые механизмы.

Криптографическая подпись результата: целостность можно проверить самому. Сертификаты вместо оценок: не «этот агент хороший», а «этот агент прошёл верификацию для финансовых операций». Zero-knowledge proofs (доказательство без раскрытия данных): система подтверждает корректность, не показывая приватные данные.

Разница примерно как между кредитным рейтингом и блокчейн-верификацией. Первый просит верить, вторая даёт проверить.

## Проверка результатов

Выход агента тоже не принимается на веру. Для операций вроде «обновил конфиг в K8s» или «запустил миграцию БД» ошибка оставляет систему в broken state.

Система сверяет результат со сценарием успеха и учитывает заявленную уверенность агента («уверен на 95%» против «не знаю»). Если валидация не прошла, включается fallback.

В финтехе это жёстче всего: платёж не бывает «вроде обработан». Либо обработан с доказательством, либо откачен.

## Передача между агентами

Задача может пройти через нескольких агентов: A парсит JSON, B валидирует, C пишет в БД. При каждой передаче система фиксирует три вещи. Кто отвечает за результат, если B потеряет часть данных. Какие права переходят вместе с задачей: может ли A заменить X на Y, может ли B откатить работу C. И кто вправе остановить цепочку посередине.

Логика та же, что в микросервисах: если сервис упал, кто-то должен инициировать откат. Разница только в том, что участники — агенты.

## Иерархия доступа

Уровень 0 — только чтение. Агент смотрит логи и метрики, ничего не меняет.

Уровень 1 — предложения. Готовит изменение и ждёт подтверждения.

Уровень 2 — исполнение с откатом. Запускает сам, откат доступен в течение 5 секунд.

Уровень 3 — автономная работа с логированием. Полные права, всё пишется в журнал.

Уровень 4 — критические операции: деньги, удаление, безвозвратные изменения. Нужно согласие двух агентов.

Права пересчитываются на каждой операции. Сломал что-то на уровне 3 — вернулся на уровень 2.

## Как система оценивает готовность

Процент успеха на последних N операциях: если агент прав в 98% похожих задач, доверие растёт.

Стоимость ошибки: чтение конфига дёшево, удаление данных дорого, вес у них разный.

Заявленная уверенность агента: «уверен на 70%» понижает уровень доступа.

Консистентность по времени: если в пиковые часы агент ошибается чаще, его ограничивают по времени работы.

## Что может сломаться

Каскадный отказ. A передал задачу B, B передал C, C упал, цепочка рушится. Лечится контрольной точкой у каждого агента: можно вернуться на несколько шагов назад.

Злонамеренный (византийский) отказ. Агент сломан и отдаёт другим заведомо неверные данные, а не просто ошибается сам. Лечится согласием большинства агентов.

Исчерпание ресурсов. Агент застрял на задаче и выедает ресурсы. Лечится жёстким timeout и автоматическим завершением при превышении лимитов.

Дрейф прав доступа. Агенту выдали права, он ими не пользовался, про них забыли — а потом он их неожиданно применил. Лечится регулярным переподтверждением прав, например раз в неделю.

## Видимость системы и мониторинг

Если не видно, что делает AI, системой не управляют.

Логируется весь жизненный цикл задачи: кто отдал, кому, с какими правами; промежуточные этапы, если задача разбита на подзадачи; финальный результат и его валидация; время выполнения и потраченные ресурсы.

Уверенность выводится графиком: «был уверен на 90%, потом сомнения упали до 40%» — это сигнал проблемы.

Отклонения ловятся сравнением текущего поведения агента с историческим: стал ошибаться чаще или работать медленнее — поднимается алерт.

Каждое решение агента записано и доступно для проверки, для аудита и разбора инцидентов.

## Где это работает на практике

### DevOps и инфраструктура

Агент автоматизирует развёртывание. Начинаем с уровня 1 (предложения), поднимаемся до уровня 2 (исполнение с откатом за 5 секунд), затем до уровня 3. И то только в staging: в production нужно человеческое подтверждение. Отслеживаем уверенность и отклонения, начал ошибаться чаще — урезаем права.

### Финансовые пайплайны данных

Агент обрабатывает денежные операции. Удалить запись он не может, только пометить как «в обработке». Валидация на каждом шаге, при ошибке откат и алерт. Многоагентный сценарий: парсинг на уровне 1, проверка на уровне 2, сохранение на уровне 3 и только при согласии двоих.

### Безопасность и аудит логов

Агент мониторит логи и ищет аномалии, по умолчанию на уровне 0. Нашёл подозрительную активность — передаёт выше и привлекает второго агента для независимой проверки. Всё пишется в журнал.

### Оркестрация нескольких агентов

Несколько агентов работают над одной задачей. Фреймворк нужен, чтобы они не мешали друг другу и честно откатывались при сбое. Каскадный отказ не должен положить всю систему, поэтому каждый агент хранит контрольную точку.

Фреймворк Google DeepMind описывает не «как я дал команду в Slack», а как строится распределённая система, где люди и агенты работают вместе, никто не может всё сломать, и по журналу всегда видно, кто что сделал и почему.

---

*Источник: [arxiv.org/abs/2602.11865](https://arxiv.org/abs/2602.11865) — Google DeepMind, «Intelligent AI Delegation». Разбор на [vc.ru](https://vc.ru/ai/3018438-razbor-delegirovaniya-zadach-ai-v-google-deepmind).*

*Дисклеймер / Disclaimer: material is published for informational and research purposes. [Полный отказ от ответственности / Full disclaimer](https://notes.kazakov.xyz/legal/disclaimer/).*
