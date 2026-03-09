# UI Design System

## Principles
- 简洁
- 一致
- 高效
- 可信

## Apps
- admin-web: 高密度后台，表格优先
- institution-web: 看板与图表
- assessee-web: PC 答题，居中专注
- assessee-h5: 移动答题，单列大触控
- portal-web: 展示型，更多留白

## Core Tokens
### Colors
- primary: #4361EE
- primary-light: #EEF1FD
- primary-dark: #2D4BC7
- success: #22C55E
- warning: #F59E0B
- error: #EF4444
- info: #3B82F6
- text-primary: #1A1A2E
- text-secondary: #6B7280
- border: #E5E7EB
- bg-page: #F5F7FA
- bg-card: #FFFFFF

### Typography
- h1: 32/1.3/300
- h2: 24/1.4/300
- h3: 20/1.4/400
- h4: 18/1.5/400
- body-lg: 16/1.6/400
- body: 14/1.6/400
- body-sm: 13/1.6/400
- caption: 12/1.5/400

### Spacing
- 4, 8, 12, 16, 24, 32, 48, 64

### Radius
- 4, 8, 12, 16, full

### Shadow
- xs, sm, md, lg

## Components
- Button: primary / secondary / text / danger / disabled
- Form: clear labels, required mark, inline error
- Table: hover row, clear states, unified empty value
- Badge: success / warning / error / info / primary / neutral
- Toast: short, clear, semantic

## Motion
- hover: 100ms
- expand: 200ms
- route: 250ms
- modal: 300ms
- easing-in: ease-in
- easing-out: ease-out
- easing-switch: ease-in-out

## AI Rules
- 优先复用 token
- 相同语义不得换样式
- 可替换 style profile
- 不可破坏 foundation rules