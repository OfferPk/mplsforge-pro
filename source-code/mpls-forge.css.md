:root {  
  \--bg-primary: \#070b14;  
  \--bg-card: \#0d1527;  
  \--bg-card-hover: \#121c33;  
  \--border-color: \#1e2d4d;  
  \--text-main: \#f1f5f9;  
  \--text-muted: \#94a3b8;  
  \--accent-cyan: \#06b6d4;  
  \--accent-teal: \#14b8a6;  
  \--accent-blue: \#3b82f6;  
  \--accent-purple: \#8b5cf6;  
  \--success: \#10b981;  
  \--warning: \#f59e0b;  
  \--danger: \#ef4444;  
  \--font-sans: \-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;  
  \--font-mono: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;  
}

\* {  
  box-sizing: border-box;  
  margin: 0;  
  padding: 0;  
}

body {  
  background-color: var(--bg-primary);  
  color: var(--text-main);  
  font-family: var(--font-sans);  
  line-height: 1.5;  
  padding: 24px;  
}

.studio-container {  
  max-width: 1400px;  
  margin: 0 auto;  
}

/\* Header \*/  
.studio-header {  
  display: flex;  
  justify-content: space-between;  
  align-items: center;  
  padding-bottom: 20px;  
  border-bottom: 1px solid var(--border-color);  
  margin-bottom: 24px;  
}

.brand-area {  
  display: flex;  
  align-items: center;  
  gap: 14px;  
}

.brand-logo {  
  width: 44px;  
  height: 44px;  
  background: linear-gradient(135deg, var(--accent-cyan), var(--accent-purple));  
  border-radius: 10px;  
  display: flex;  
  align-items: center;  
  justify-content: center;  
  font-weight: 900;  
  font-size: 20px;  
  color: \#fff;  
  box-shadow: 0 0 16px rgba(6, 182, 212, 0.4);  
}

.brand-titles h1 {  
  font-size: 24px;  
  font-weight: 800;  
  background: linear-gradient(90deg, \#38bdf8, \#818cf8);  
  \-webkit-background-clip: text;  
  \-webkit-text-fill-color: transparent;  
}

.brand-titles p {  
  font-size: 13px;  
  color: var(--text-muted);  
}

.header-badges {  
  display: flex;  
  gap: 10px;  
}

.badge {  
  display: inline-flex;  
  align-items: center;  
  padding: 4px 10px;  
  border-radius: 6px;  
  font-size: 12px;  
  font-weight: 600;  
  text-transform: uppercase;  
  letter-spacing: 0.5px;  
}

.badge-cyan { background: rgba(6, 182, 212, 0.15); color: \#38bdf8; border: 1px solid rgba(6, 182, 212, 0.3); }  
.badge-purple { background: rgba(139, 92, 246, 0.15); color: \#c084fc; border: 1px solid rgba(139, 92, 246, 0.3); }  
.badge-green { background: rgba(16, 185, 129, 0.15); color: \#34d399; border: 1px solid rgba(16, 185, 129, 0.3); }  
.badge-red { background: rgba(239, 68, 68, 0.15); color: \#f87171; border: 1px solid rgba(239, 68, 68, 0.3); }

/\* Grid Layout \*/  
.studio-grid {  
  display: grid;  
  grid-template-columns: 460px 1fr;  
  gap: 24px;  
}

@media (max-width: 1080px) {  
  .studio-grid {  
    grid-template-columns: 1fr;  
  }  
}

.panel-card {  
  background-color: var(--bg-card);  
  border: 1px solid var(--border-color);  
  border-radius: 12px;  
  padding: 20px;  
  margin-bottom: 20px;  
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.4);  
}

.panel-title {  
  font-size: 15px;  
  font-weight: 700;  
  color: \#e2e8f0;  
  margin-bottom: 16px;  
  display: flex;  
  justify-content: space-between;  
  align-items: center;  
  border-bottom: 1px solid rgba(255, 255, 255, 0.06);  
  padding-bottom: 10px;  
}

/\* Forms \*/  
.form-group {  
  margin-bottom: 14px;  
}

.form-group label {  
  display: block;  
  font-size: 12px;  
  font-weight: 600;  
  color: var(--text-muted);  
  margin-bottom: 5px;  
}

.form-row {  
  display: grid;  
  grid-template-columns: 1fr 1fr;  
  gap: 12px;  
}

.form-row-3 {  
  display: grid;  
  grid-template-columns: 1fr 1fr 1fr;  
  gap: 10px;  
}

input, select {  
  width: 100%;  
  padding: 8px 12px;  
  background-color: \#070c18;  
  border: 1px solid var(--border-color);  
  border-radius: 6px;  
  color: var(--text-main);  
  font-size: 13px;  
  font-family: inherit;  
  outline: none;  
  transition: border-color 0.2s;  
}

input:focus, select:focus {  
  border-color: var(--accent-cyan);  
}

.checkbox-label {  
  display: flex;  
  align-items: center;  
  gap: 8px;  
  font-size: 13px;  
  cursor: pointer;  
  user-select: none;  
  color: \#cbd5e1;  
}

.checkbox-label input\[type="checkbox"\] {  
  width: 16px;  
  height: 16px;  
  accent-color: var(--accent-cyan);  
}

/\* Label Stack Visualizer \*/  
.label-stack-list {  
  display: flex;  
  flex-direction: column;  
  gap: 8px;  
  margin-bottom: 14px;  
}

.lse-card {  
  background: \#091020;  
  border: 1px solid var(--border-color);  
  border-radius: 8px;  
  padding: 10px;  
  display: grid;  
  grid-template-columns: 32px 1fr 80px 70px 70px 32px;  
  gap: 8px;  
  align-items: center;  
}

.lse-index {  
  font-size: 12px;  
  font-weight: 800;  
  color: var(--accent-cyan);  
  text-align: center;  
}

.btn-remove-lse {  
  background: transparent;  
  border: none;  
  color: \#94a3b8;  
  cursor: pointer;  
  font-size: 16px;  
  display: flex;  
  align-items: center;  
  justify-content: center;  
}

.btn-remove-lse:hover {  
  color: var(--danger);  
}

.btn {  
  background: linear-gradient(135deg, \#0284c7, \#0369a1);  
  color: \#fff;  
  border: none;  
  padding: 8px 16px;  
  border-radius: 6px;  
  font-weight: 600;  
  font-size: 13px;  
  cursor: pointer;  
  display: inline-flex;  
  align-items: center;  
  gap: 6px;  
  transition: opacity 0.2s;  
}

.btn:hover { opacity: 0.9; }

.btn-secondary {  
  background: \#1e293b;  
  border: 1px solid var(--border-color);  
  color: \#cbd5e1;  
}  
.btn-secondary:hover { background: \#334155; }

/\* MTU Stat Cards \*/  
.mtu-metrics {  
  display: grid;  
  grid-template-columns: repeat(4, 1fr);  
  gap: 12px;  
  margin-bottom: 16px;  
}

.metric-box {  
  background: \#060b17;  
  border: 1px solid var(--border-color);  
  border-radius: 8px;  
  padding: 12px;  
  text-align: center;  
}

.metric-val {  
  font-size: 20px;  
  font-weight: 800;  
  font-family: var(--font-mono);  
  color: \#38bdf8;  
}

.metric-val.alert-critical { color: var(--danger); }  
.metric-val.alert-warn { color: var(--warning); }  
.metric-val.alert-ok { color: var(--success); }

.metric-lbl {  
  font-size: 11px;  
  color: var(--text-muted);  
  text-transform: uppercase;  
  margin-top: 4px;  
}

/\* Audit Score Banner \*/  
.audit-score-banner {  
  display: flex;  
  align-items: center;  
  gap: 20px;  
  padding: 16px;  
  background: \#060b17;  
  border: 1px solid var(--border-color);  
  border-radius: 10px;  
  margin-bottom: 18px;  
}

.grade-circle {  
  width: 64px;  
  height: 64px;  
  border-radius: 50%;  
  display: flex;  
  align-items: center;  
  justify-content: center;  
  font-size: 26px;  
  font-weight: 900;  
  border: 3px solid;  
}

.grade-A, .grade-A\\+ { border-color: var(--success); color: var(--success); background: rgba(16, 185, 129, 0.1); }  
.grade-B { border-color: \#3b82f6; color: \#3b82f6; background: rgba(59, 130, 246, 0.1); }  
.grade-C { border-color: var(--warning); color: var(--warning); background: rgba(245, 158, 11, 0.1); }  
.grade-D, .grade-F { border-color: var(--danger); color: var(--danger); background: rgba(239, 68, 68, 0.1); }

.audit-meta h3 { font-size: 17px; font-weight: 700; color: \#f8fafc; }  
.audit-meta p { font-size: 12px; color: var(--text-muted); margin-top: 4px; }

/\* Audit Checklist \*/  
.audit-list {  
  display: flex;  
  flex-direction: column;  
  gap: 8px;  
}

.audit-item {  
  background: \#080f20;  
  border: 1px solid rgba(255, 255, 255, 0.05);  
  border-radius: 6px;  
  padding: 10px 14px;  
  display: grid;  
  grid-template-columns: 28px 1fr 60px;  
  align-items: center;  
  gap: 12px;  
}

.audit-icon {  
  font-size: 16px;  
  text-align: center;  
}

.audit-info h4 { font-size: 13px; font-weight: 600; color: \#e2e8f0; }  
.audit-info p { font-size: 11px; color: var(--text-muted); margin-top: 2px; }  
.audit-pts { font-family: var(--font-mono); font-size: 12px; font-weight: 700; text-align: right; }

/\* Tabs & Code viewer \*/  
.tabs-nav {  
  display: flex;  
  gap: 8px;  
  border-bottom: 1px solid var(--border-color);  
  padding-bottom: 10px;  
  margin-bottom: 12px;  
}

.tab-btn {  
  background: transparent;  
  border: 1px solid transparent;  
  color: var(--text-muted);  
  padding: 6px 12px;  
  border-radius: 6px;  
  font-size: 12px;  
  font-weight: 600;  
  cursor: pointer;  
}

.tab-btn.active {  
  background: \#1e293b;  
  border-color: var(--accent-cyan);  
  color: \#38bdf8;  
}

.code-viewer {  
  background: \#040711;  
  border: 1px solid var(--border-color);  
  border-radius: 8px;  
  padding: 16px;  
  font-family: var(--font-mono);  
  font-size: 12px;  
  line-height: 1.6;  
  color: \#cbd5e1;  
  overflow-x: auto;  
  white-space: pre;  
  max-height: 380px;  
}

.code-header-actions {  
  display: flex;  
  justify-content: flex-end;  
  margin-bottom: 8px;  
}  
