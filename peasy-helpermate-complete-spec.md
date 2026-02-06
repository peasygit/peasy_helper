# Peasy / HelperMate 完整平台開發規格書

**版本**: v1.0  
**日期**: 2026年2月5日  
**目標**: 家務 HR／外傭配對平台 Web MVP

---

## 目錄

1. [項目概述](#項目概述)
2. [市場定位與競品差異](#市場定位與競品差異)
3. [技術架構](#技術架構)
4. [資料模型設計](#資料模型設計)
5. [核心功能模組](#核心功能模組)
6. [配對邏輯（含五行／星座）](#配對邏輯含五行星座)
7. [Design System（HSBC 風格）](#design-system-hsbc-風格)
8. [開發階段與時程](#開發階段與時程)
9. [API 端點規格](#api-端點規格)
10. [表單欄位清單](#表單欄位清單)

---

## 項目概述

### 願景

打造一個「家務 HR 顧問」平台，不只是配對外傭與僱主，而是幫助家庭做完整的人力規劃、流程管理、長期關係維護，專注於年輕家庭與長者照顧市場。

### 核心價值主張

- **僱主端**: 3分鐘問卷 → 精選推薦 → 全流程協助（面試、文件、續約提醒）
- **外傭端**: 完全免費、無債務壓力、公平對待、清晰職涯建議
- **差異化**: 深度 HR workflow + 長者照顧整合 + 五行／星座微調配對

---

## 市場定位與競品差異

### 主要競品: HelperPlace

| 面向 | HelperPlace | Peasy / HelperMate |
|------|-------------|-------------------|
| **定位** | 外傭直聘平台（classified + marketplace） | 家務 HR 顧問（end-to-end service） |
| **客群** | 有時間自己搜尋的僱主 | 忙碌年輕家庭 + 長者照顧家庭 |
| **商業模式** | 時段解鎖資料庫（7/30/60日一次性付費） | 配對費 + 可選月費（持續 HR 服務） |
| **功能重點** | 搜尋、訊息、job posting | 需求分析、精選推薦、文件自動化、續約管理、長者 monitoring |
| **技術層** | 較輕 operation | 深度 automation（WhatsApp bot、AppSheet、Make.com） |
| **用戶數** | 260,000+ 用戶 | 從零開始，錯位競爭 |

### 品牌定位

**HelperPlace**: 「No middleman」、「Fair recruitment」、「Ethical platform」  
**Peasy / HelperMate**: 「家務 HR，為你照顧『人』和『家』」、「不只找外傭，更是家庭人力顧問」

---

## 技術架構

### 整體分層

```
┌─────────────────────────────────────┐
│     前端 Web App (Next.js/React)     │
│  - 僱主端、外傭端、Admin 後台         │
│  - i18n (中英)、Responsive Design   │
└──────────────┬──────────────────────┘
               │ REST API
┌──────────────▼──────────────────────┐
│     後端 API (Node.js / FastAPI)     │
│  - 認證 (JWT)、配對邏輯、通知觸發    │
│  - 五行／星座計算模組                │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   資料庫 (PostgreSQL)                │
│  - Users、Employers、Helpers、Jobs   │
│  - Applications/Matches、Logs        │
└─────────────────────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   外部服務整合                       │
│  - Stripe (收費)                     │
│  - WhatsApp Business API (通知)     │
│  - S3/Supabase Storage (文件)       │
└─────────────────────────────────────┘
```

### 技術堆疊建議

- **前端**: Next.js 14+ (App Router) + TypeScript + Tailwind CSS
- **後端**: Node.js (NestJS) 或 Python (FastAPI)
- **資料庫**: PostgreSQL (Supabase / AWS RDS)
- **ORM**: Prisma (Node.js) 或 SQLAlchemy (Python)
- **認證**: JWT + Bcrypt
- **部署**: Vercel (前端) + Render/Railway (後端)
- **CI/CD**: GitHub Actions

---

## 資料模型設計

### 核心資料表

#### 1. Users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(20),
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(20) NOT NULL, -- 'employer' | 'helper' | 'admin'
  status VARCHAR(20) DEFAULT 'active', -- 'active' | 'suspended'
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### 2. Employers
```sql
CREATE TABLE employers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  name VARCHAR(100),
  household_size INT,
  has_children BOOLEAN DEFAULT FALSE,
  children_ages TEXT, -- JSON array
  has_elderly BOOLEAN DEFAULT FALSE,
  elderly_care_needs TEXT, -- JSON array
  location VARCHAR(100),
  language_preferences TEXT, -- JSON array
  preferred_helper_traits TEXT, -- JSON array
  household_rules TEXT,
  preferred_start_date DATE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### 3. Helpers
```sql
CREATE TABLE helpers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  full_name VARCHAR(100) NOT NULL,
  display_name VARCHAR(50), -- 對外顯示名
  nationality VARCHAR(50),
  birthdate DATE,
  religion VARCHAR(50),
  current_location VARCHAR(100),
  contract_status VARCHAR(50), -- 'finishing' | 'transfer' | 'freelance' | 'overseas'
  available_from DATE,
  years_experience_total INT,
  years_experience_local INT, -- HK/SG
  education_level VARCHAR(50),
  languages TEXT, -- JSON: [{"lang": "english", "level": "fluent"}]
  about_me TEXT,
  profile_photo_url TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### 4. HelperSkills
```sql
CREATE TABLE helper_skills (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  helper_id UUID REFERENCES helpers(id) ON DELETE CASCADE,
  skill_type VARCHAR(50), -- 'cooking_chinese' | 'cleaning' | 'driving' | 'gardening'
  proficiency_level VARCHAR(20), -- 'basic' | 'intermediate' | 'advanced'
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### 5. HelperCareExperience
```sql
CREATE TABLE helper_care_experience (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  helper_id UUID REFERENCES helpers(id) ON DELETE CASCADE,
  target_type VARCHAR(50), -- 'infant' | 'toddler' | 'child' | 'teenager' | 'healthy_elderly' | 'disabled_elderly'
  years_experience INT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### 6. HelperProfileMeta（五行／星座）
```sql
CREATE TABLE helper_profile_meta (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  helper_id UUID REFERENCES helpers(id) ON DELETE CASCADE,
  western_zodiac VARCHAR(20), -- 'aries' | 'taurus' | ...
  wuxing_element VARCHAR(10), -- 'wood' | 'fire' | 'earth' | 'metal' | 'water'
  wuxing_calculation_data JSONB, -- 完整八字資料（日後擴展）
  personality_traits TEXT, -- JSON array
  work_style_preference VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### 7. Jobs
```sql
CREATE TABLE jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  employer_id UUID REFERENCES employers(id) ON DELETE CASCADE,
  title VARCHAR(200),
  description TEXT,
  duties JSONB, -- 勾選項目：['cooking', 'cleaning', 'childcare', 'eldercare']
  preferred_experience_years INT,
  preferred_languages TEXT, -- JSON array
  preferred_start_date DATE,
  salary_range VARCHAR(50),
  status VARCHAR(20) DEFAULT 'active', -- 'active' | 'filled' | 'closed'
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### 8. Applications / Matches
```sql
CREATE TABLE matches (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id UUID REFERENCES jobs(id) ON DELETE CASCADE,
  helper_id UUID REFERENCES helpers(id) ON DELETE CASCADE,
  source_type VARCHAR(20), -- 'auto_match' | 'helper_applied' | 'admin_recommended'
  match_score DECIMAL(5,2), -- 0-100
  match_breakdown JSONB, -- 詳細分數組成
  status VARCHAR(20) DEFAULT 'pending', -- 'pending' | 'shortlisted' | 'interviewed' | 'hired' | 'rejected'
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(job_id, helper_id)
);
```

#### 9. Events / Logs
```sql
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  event_type VARCHAR(50), -- 'view_profile' | 'submit_questionnaire' | 'contact_helper'
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 核心功能模組

### 模組 1: 僱主需求收集（家務 HR 問卷）

#### 問卷欄位分類

**一、基本家庭資料**
1. 家庭人數（成人 / 小朋友 / 長者）
2. 主要語言（廣東話、普通話、英語、其他）
3. 居住環境（房間數、有無樓梯、露台、寵物）

**二、工作重點與生活節奏**
4. 最需要處理的三項工作（多選）
5. 平日在家情況（全日有人 / 早上 / 晚上 / 沒人）
6. 最重視的兩項特質（耐性、整齊、主動、安靜等）

**三、小朋友／長者特別需要**
7. 是否有特別照顧需求（嬰兒、幼兒、學前、行動不便長者等）
8. 照顧範圍（扶抱、沖涼、餵藥、陪診、運動、功課輔導）

**四、價值觀與相處方式**
9. 相處方式偏好（像同事 / 像家人 / 介乎中間）
10. 家中底線規矩（不可玩手機帶小孩、不可帶朋友、宗教飲食規定）

**五、實際條件**
11. 預計開始時間（1個月內 / 3個月內 / 更遠）
12. 經驗期望（接受新人 / 需有相似背景 / 必須本地經驗X年）
13. 服務類型（僅配對推薦 / 需全程協助）

**六、五行／星座（可選填）**
14. 僱主／主要家庭成員生日（用於能量配對）

#### 表單體驗設計
- 單頁 form，視覺分 5 個 sections
- 進度條顯示：「第 X 步，共 5 步」
- 多選用 checkbox，單選用 radio，日期用 date picker
- 提交後顯示：「已收到，48 小時內提供 3–5 位精選候選」

---

### 模組 2: 外傭資料收集

#### 必填欄位（第一版）

**基本身份**
- 姓名（full_name + display_name）
- 國籍（dropdown）
- 出生年份或年齡區間
- 宗教（選填）

**聯絡與狀態**
- 現居地（HK / SG / 海外）
- 合約狀態（約滿 / 提早完約 / 轉工 / 本地自由身 / 海外）
- 可開始工作日期

**工作與經驗**
- 總工作年數
- 本地（HK/SG）工作年數
- 曾服務地區（多選）
- 服務過幾個家庭

**技能與照顧對象**
- 技能勾選（煮食細分中西菜、清潔、洗燙、買餸、園藝、開車）
- 照顧經驗（嬰兒、幼兒、小學生、長者、病人，多選）
- 語言能力（英文、廣東話、普通話，各標註程度）

**教育與證書**
- 最高學歷（小學 / 中學 / 大專）
- 相關訓練證書（急救 / 照護 / 烹飪 / 駕駛）

#### 第二層：性格與偏好

- 工作偏好對象（優先次序）
- 休息安排接受度
- 住屋安排接受度
- 3個形容自己的詞（dropdown多選）
- 期望相處方式
- 期望薪金區間
- 不能接受的工作類型

#### 五行／星座欄位

- 出生日期（自動計算 western_zodiac）
- 出生時間（選填，可用三段：早上 / 下午 / 晚上）
- 系統內部計算 wuxing_element（木火土金水）

---

### 模組 3: 配對引擎

#### 配對分數公式（0–100）

```
match_score = 
  base_skill_score (40%) +
  experience_score (20%) +
  preference_score (15%) +
  language_score (10%) +
  availability_score (5%) +
  element_bonus (5%) +
  zodiac_bonus (5%)
```

#### 各項計分邏輯

**1. base_skill_score (0–40分)**
- 比對 job.duties 與 helper_skills.skill_type
- 完全匹配的技能 +10 分
- 部分匹配（如 cooking_chinese vs cooking_western）+5 分
- 照顧對象匹配（job 需要 childcare，helper 有 child 經驗）+10 分

**2. experience_score (0–20分)**
- 總年資符合預期：+10 分
- 本地經驗符合預期：+10 分
- 若 helper 年資遠超需求，略減分（避免 overqualified 不穩定）

**3. preference_score (0–15分)**
- 比對僱主「最重視特質」與 helper「性格 traits」
- 每個匹配項 +5 分（最多 3 項）

**4. language_score (0–10分)**
- 語言完全匹配 +10 分
- 部分匹配 +5 分
- 不匹配但可基本溝通 +2 分

**5. availability_score (0–5分)**
- helper.available_from 在 job.preferred_start_date 前後 1 個月內：+5 分
- 超過 3 個月：+2 分

**6. element_bonus (0–5分) — 五行微調**
- 使用相生循環表：水→木→火→土→金→水
- 僱主 element → helper element 符合相生：+5 分
- 同 element：+3 分
- 相剋（如金剋木）：-2 分（不排除，只降低）

五行相生表：
```
Water → Wood → Fire → Earth → Metal → Water
水生木、木生火、火生土、土生金、金生水
```

五行相剋表：
```
Metal → Wood, Wood → Earth, Earth → Water, Water → Fire, Fire → Metal
金剋木、木剋土、土剋水、水剋火、火剋金
```

**7. zodiac_bonus (0–5分) — 星座微調**
- 同元素星座（火配火、土配土、風配風、水配水）：+5 分
- 互補元素（火配風、土配水）：+3 分
- 不匹配：0 分

星座元素表：
```
Fire: Aries, Leo, Sagittarius
Earth: Taurus, Virgo, Capricorn
Air: Gemini, Libra, Aquarius
Water: Cancer, Scorpio, Pisces
```

#### 排序與呈現

1. 計算所有符合基本條件（地區、合約狀態、可開始日期）的 helpers
2. 對每個 helper 計算 match_score
3. 排序：match_score DESC
4. 取前 3–10 名作為推薦列表
5. Admin 可手動調整（加減分、標記「人工推薦」）

---

### 模組 4: 前端頁面結構

#### 首頁（Home）
- Hero 區塊：大標「家務 HR，為你照顧『人』和『家』」+ CTA
- 三大賣點卡片
- 流程 4 步驟
- 年輕家庭 vs 長者照顧兩欄
- Footer

#### 僱主問卷頁（/employers/questionnaire）
- Stepper（5 步進度條）
- 每步 3–5 條問題
- 提交後確認頁

#### 外傭註冊頁（/helpers/register）
- Stepper（3–4 步）
- 基本資料 → 技能經驗 → 性格偏好 → 完成
- 提交後顯示「個人檔案預覽」

#### Dashboard — 僱主端（/employers/dashboard）
- 左側：我的需求摘要
- 右側：推薦候選人列表（卡片式）
- 點擊進入詳情頁

#### Dashboard — 外傭端（/helpers/dashboard）
- 我的檔案預覽
- 推薦給我的 jobs
- 我申請的 jobs

#### 候選人詳情頁（/helpers/:id）
- Header：名字、國籍、Match Score Badge
- Sections：基本資料 / 工作經驗 / 技能 / 性格與能量 / 聯絡（付費後解鎖）

#### Admin 後台（/admin）
- 用戶列表（Employers / Helpers）
- 配對管理（手動調整 score、標記推薦）
- 簡單報表（註冊數、配對成功率）

---

## Design System (HSBC 風格)

### 設計原則

參考 HSBC Create Design System 的核心理念：
- **Art of simplicity**: 少量顏色、清晰字體、充足留白
- **Accessible by design**: WCAG AA 級別、清晰 focus 狀態
- **HSBC in motion**: 低調動效、supportive not distracting
- **Creative Hexagons**: 幾何造型靈感（但不直接複製六角標誌）

### Foundations

#### 顏色系統

```css
/* Primary */
--color-primary: #DB0011; /* 深紅（參考 HSBC，調整避免完全相同） */
--color-primary-hover: #B00010;
--color-primary-active: #8B000D;

/* Neutral */
--color-neutral-50: #FAFAFA;
--color-neutral-100: #F5F5F5;
--color-neutral-200: #EEEEEE;
--color-neutral-300: #E0E0E0;
--color-neutral-400: #BDBDBD;
--color-neutral-500: #9E9E9E;
--color-neutral-700: #616161;
--color-neutral-900: #212121;

/* Semantic */
--color-success: #4CAF50;
--color-warning: #FF9800;
--color-error: #F44336;
--color-info: #2196F3;

/* Text */
--color-text-primary: var(--color-neutral-900);
--color-text-secondary: var(--color-neutral-700);
--color-text-disabled: var(--color-neutral-400);
```

#### 字體層級

```css
/* Font Family */
--font-family: 'Inter', 'Noto Sans TC', -apple-system, sans-serif;

/* Heading Scale */
--font-h1: 32px / 1.2 / 600; /* size / line-height / weight */
--font-h2: 24px / 1.3 / 600;
--font-h3: 20px / 1.4 / 600;
--font-h4: 18px / 1.4 / 600;

/* Body Scale */
--font-body-lg: 16px / 1.5 / 400;
--font-body: 14px / 1.5 / 400;
--font-body-sm: 12px / 1.5 / 400;
--font-caption: 11px / 1.4 / 400;
```

#### 間距系統

```css
--spacing-xs: 4px;
--spacing-sm: 8px;
--spacing-md: 16px;
--spacing-lg: 24px;
--spacing-xl: 32px;
--spacing-2xl: 48px;
--spacing-3xl: 64px;
```

#### Grid & Breakpoints

```css
/* Desktop: 12-column grid, max-width 1280px */
/* Tablet: 8-column, 768-1024px */
/* Mobile: 4-column, 375-767px */

--breakpoint-sm: 640px;
--breakpoint-md: 768px;
--breakpoint-lg: 1024px;
--breakpoint-xl: 1280px;
```

### 組件規格

#### Button

**Variants**:
- `primary`: 實心紅底白字
- `secondary`: 白底紅字，紅色邊框
- `tertiary`: 純文字，無邊框

**Sizes**:
- `sm`: padding 8px 16px, font 14px
- `md`: padding 12px 24px, font 16px
- `lg`: padding 16px 32px, font 18px

**States**:
- Default / Hover / Active / Disabled / Focus

**Props** (React):
```typescript
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'tertiary';
  size: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  loading?: boolean;
  icon?: ReactNode;
  children: ReactNode;
  onClick?: () => void;
}
```

#### Input / TextField

**Variants**:
- `text` | `email` | `password` | `tel` | `number` | `date`

**States**:
- Default / Focus / Error / Disabled

**Props**:
```typescript
interface InputProps {
  type: string;
  label: string;
  placeholder?: string;
  helperText?: string;
  errorText?: string;
  required?: boolean;
  disabled?: boolean;
  value: string;
  onChange: (value: string) => void;
}
```

**樣式要點**:
- Label 在上方
- 邊框 1px solid neutral-300
- Focus 時邊框變 primary color
- Error 時邊框變 error color + 下方顯示 errorText

#### Select

與 Input 類似，但用 dropdown icon（向下箭頭）

#### Radio / Checkbox

- Radio: 圓形選擇器，選中時內圈填充 primary color
- Checkbox: 方形，選中時顯示勾號

#### Card

**樣式**:
```css
background: white;
border: 1px solid var(--color-neutral-200);
border-radius: 8px;
padding: var(--spacing-lg);
box-shadow: 0 1px 3px rgba(0,0,0,0.08);
```

**Hover** (when clickable):
```css
box-shadow: 0 4px 12px rgba(0,0,0,0.12);
border-color: var(--color-neutral-300);
```

#### Tag / Badge

**樣式**:
```css
display: inline-flex;
padding: 4px 12px;
border-radius: 16px;
font-size: 12px;
font-weight: 500;
```

**Variants**:
- `default`: background neutral-200, text neutral-900
- `success`: background success-light, text success-dark
- `warning`: background warning-light, text warning-dark
- `primary`: background primary-light, text primary

#### Modal

**結構**:
- Overlay (半透明黑背景)
- Container (白色卡片，中央顯示)
- Header (標題 + 關閉按鈕)
- Body (內容)
- Footer (按鈕組)

**尺寸**:
- `sm`: max-width 400px
- `md`: max-width 600px
- `lg`: max-width 800px

#### Stepper

**樣式**:
- 水平線連接圓點
- 已完成步驟：圓點填充 primary，線為 primary
- 當前步驟：圓點邊框 primary，內部白
- 未完成步驟：灰色

**結構**:
```
[1 基本資料] ── [2 技能] ── [3 性格] ── [4 完成]
    (紅)       (灰線)    (灰)      (灰)
```

#### Alert / Banner

**Variants**:
- `success` / `warning` / `error` / `info`

**結構**:
- Icon (左側) + Message (中間) + Close button (右側，可選)

**樣式**:
```css
padding: 12px 16px;
border-radius: 8px;
border-left: 4px solid [variant-color];
background: [variant-color-light];
```

### 頁面模板

#### Home Page Template

```
┌─────────────────────────────────────────┐
│ Navbar (Logo + Menu + Login/Signup)    │
├─────────────────────────────────────────┤
│ Hero Section                             │
│ [左：標題 + 副標 + CTA] [右：圖片]         │
├─────────────────────────────────────────┤
│ 3 Columns Cards (賣點)                   │
│ [卡片1] [卡片2] [卡片3]                   │
├─────────────────────────────────────────┤
│ Process Section (4 Steps)               │
│ [步驟1] → [步驟2] → [步驟3] → [步驟4]     │
├─────────────────────────────────────────┤
│ 2 Columns (年輕家庭 vs 長者照顧)          │
│ [左欄] [右欄]                            │
├─────────────────────────────────────────┤
│ CTA Banner + Footer                     │
└─────────────────────────────────────────┘
```

#### Questionnaire Page Template

```
┌─────────────────────────────────────────┐
│ Navbar                                   │
├─────────────────────────────────────────┤
│ Stepper (進度條)                         │
├─────────────────────────────────────────┤
│ Form Section                             │
│ ┌───────────────────────────────────┐   │
│ │ Section Title                      │   │
│ │ [Input 1]                          │   │
│ │ [Input 2]                          │   │
│ │ [Select / Radio / Checkbox]        │   │
│ └───────────────────────────────────┘   │
│ [Back] [Next / Submit]                  │
└─────────────────────────────────────────┘
```

#### Dashboard Template (List View)

```
┌─────────────────────────────────────────┐
│ Navbar                                   │
├─────────┬───────────────────────────────┤
│ Sidebar │ Main Content                  │
│         │ ┌───────────────────────────┐ │
│ - Home  │ │ Page Header               │ │
│ - Jobs  │ ├───────────────────────────┤ │
│ - Msgs  │ │ [Card 1]                  │ │
│         │ │ [Card 2]                  │ │
│         │ │ [Card 3]                  │ │
│         │ └───────────────────────────┘ │
└─────────┴───────────────────────────────┘
```

---

## 開發階段與時程

### Phase 1: MVP Foundation (2–4 週)

**目標**: 基本註冊、問卷、資料展示

**工作項**:
1. 環境設置（前後端 repo、資料庫、部署流程）
2. 資料庫 schema + 第一次 migration
3. 認證系統（註冊、登入、JWT）
4. 僱主問卷頁（前端 + API）
5. 外傭註冊頁（前端 + API）
6. Admin 基礎後台（列表查看）

**交付物**:
- 僱主可填問卷，資料存入 DB
- 外傭可註冊，資料存入 DB
- Admin 可查看所有用戶

### Phase 2: 配對引擎 (4–6 週)

**目標**: 自動配對、列表展示、詳情頁

**工作項**:
1. 配對算法實作（skill / experience / preference / language / availability 基礎分數）
2. 五行／星座計算模組（birthdate → zodiac + wuxing_element）
3. 配對 bonus 計算（element_bonus + zodiac_bonus）
4. 僱主 Dashboard：推薦列表
5. 外傭 Dashboard：推薦 jobs
6. 候選人詳情頁
7. Admin 配對管理（手動調整 score）

**交付物**:
- 僱主提交問卷後，可看到 3–10 位推薦 helper
- 每個 helper 顯示 match_score 和簡短說明
- Admin 可手動標記「推薦」或調整分數

### Phase 3: 商業模式 & 收費 (2–4 週)

**目標**: 付費解鎖聯絡方式

**工作項**:
1. Plan 資料表（7日 / 30日 / 60日 access）
2. 前端「解鎖聯絡」流程
3. Stripe 整合（payment link 或 Checkout）
4. 權限控制：未付費只看基本資料，付費後看聯絡方式
5. 發票 / 收據生成

**交付物**:
- 僱主可選擇 plan，付費後解鎖 helper 聯絡方式
- 外傭永遠免費

### Phase 4: 進階功能 (4–8 週)

**目標**: WhatsApp 通知、文件管理、續約提醒

**工作項**:
1. WhatsApp Business API 整合（配對通知、面試提醒）
2. 文件上傳（helper 證書、僱主合約）
3. 重要日子提醒（試用期結束、續約、年假）
4. 長者照顧模組（Peasy ageing-tech 整合）
5. 簡單報表與 analytics

**交付物**:
- 自動化通知流程
- 完整 HR workflow 工具

---

## API 端點規格

### 認證 (Auth)

#### POST /api/auth/register
註冊新用戶

**Request Body**:
```json
{
  "email": "user@example.com",
  "phone": "+85298765432",
  "password": "securePassword123",
  "role": "employer" // or "helper"
}
```

**Response** (201):
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "role": "employer"
  },
  "token": "jwt_token_here"
}
```

#### POST /api/auth/login
登入

**Request Body**:
```json
{
  "email": "user@example.com",
  "password": "securePassword123"
}
```

**Response** (200):
```json
{
  "user": { ... },
  "token": "jwt_token_here"
}
```

#### GET /api/auth/me
取得當前用戶資訊（需 JWT）

**Response** (200):
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "role": "employer",
    "profile": { ... }
  }
}
```

---

### 僱主 (Employers)

#### POST /api/employers/questionnaire
提交家務 HR 問卷

**Request Body**:
```json
{
  "household_size": 3,
  "has_children": true,
  "children_ages": [3, 6],
  "has_elderly": false,
  "location": "Hong Kong Island",
  "language_preferences": ["cantonese", "english"],
  "top_duties": ["cooking", "cleaning", "childcare"],
  "preferred_traits": ["patient", "organized"],
  "household_rules": "No phone while supervising children",
  "preferred_start_date": "2026-04-01",
  "birthdate": "1990-05-15" // 可選
}
```

**Response** (201):
```json
{
  "job_id": "uuid",
  "message": "Questionnaire submitted. We will provide recommendations within 48 hours."
}
```

#### GET /api/employers/matches/:job_id
取得配對結果列表

**Response** (200):
```json
{
  "job": { ... },
  "matches": [
    {
      "helper_id": "uuid",
      "display_name": "Maria",
      "nationality": "Philippines",
      "years_experience": 8,
      "match_score": 87.5,
      "match_breakdown": {
        "skill": 38,
        "experience": 18,
        "preference": 12,
        "language": 10,
        "availability": 5,
        "element_bonus": 3,
        "zodiac_bonus": 1.5
      },
      "profile_summary": "Experienced in childcare and Chinese cooking..."
    },
    ...
  ]
}
```

---

### 外傭 (Helpers)

#### POST /api/helpers/profile
建立或更新外傭檔案

**Request Body**:
```json
{
  "full_name": "Maria Santos",
  "display_name": "Maria",
  "nationality": "Philippines",
  "birthdate": "1985-03-20",
  "religion": "Catholic",
  "current_location": "Hong Kong",
  "contract_status": "finishing",
  "available_from": "2026-04-15",
  "years_experience_total": 10,
  "years_experience_local": 6,
  "education_level": "High School",
  "languages": [
    {"lang": "english", "level": "fluent"},
    {"lang": "cantonese", "level": "intermediate"}
  ],
  "skills": [
    {"type": "cooking_chinese", "level": "advanced"},
    {"type": "cleaning", "level": "advanced"},
    {"type": "childcare", "level": "intermediate"}
  ],
  "care_experience": [
    {"target": "toddler", "years": 4},
    {"target": "child", "years": 6}
  ],
  "personality_traits": ["patient", "organized", "cheerful"],
  "about_me": "I am a caring and responsible helper..."
}
```

**Response** (201):
```json
{
  "helper_id": "uuid",
  "message": "Profile created successfully",
  "profile_meta": {
    "western_zodiac": "pisces",
    "wuxing_element": "water"
  }
}
```

#### GET /api/helpers/recommended-jobs
取得推薦給外傭的 jobs

**Response** (200):
```json
{
  "jobs": [
    {
      "job_id": "uuid",
      "title": "Helper needed for young family",
      "location": "Kowloon",
      "duties": ["cooking", "cleaning", "childcare"],
      "match_score": 82.0,
      "employer_summary": "Family of 4, 2 young children..."
    },
    ...
  ]
}
```

---

### Admin

#### GET /api/admin/users
列出所有用戶

**Query Params**:
- `role`: employer | helper | admin
- `status`: active | suspended
- `page`: 頁碼
- `limit`: 每頁數量

**Response** (200):
```json
{
  "users": [ ... ],
  "pagination": {
    "total": 150,
    "page": 1,
    "limit": 20,
    "pages": 8
  }
}
```

#### PATCH /api/admin/matches/:match_id
手動調整配對分數或狀態

**Request Body**:
```json
{
  "match_score_adjustment": +5, // 加減分
  "status": "admin_recommended",
  "notes": "Manually marked as top candidate"
}
```

**Response** (200):
```json
{
  "match": { ... },
  "message": "Match updated successfully"
}
```

---

## 表單欄位清單

### 僱主問卷欄位（完整版）

| 欄位名稱 | 類型 | 選項／說明 | 必填 |
|---------|------|-----------|-----|
| `household_size` | Number | 家庭人數 | ✓ |
| `adults` | Number | 成人數 | ✓ |
| `children` | Number | 小孩數 |  |
| `children_ages` | Array | 小孩年齡陣列 |  |
| `elderly` | Number | 長者數 |  |
| `language_preferences` | Multi-select | cantonese, mandarin, english, other | ✓ |
| `home_type` | Select | apartment, house, villa |  |
| `bedrooms` | Number | 房間數 |  |
| `has_stairs` | Boolean | 有無樓梯 |  |
| `has_balcony` | Boolean | 有無露台 |  |
| `has_pets` | Boolean | 有無寵物 |  |
| `pet_types` | Array | dog, cat, bird, other |  |
| `top_duties` | Multi-select (max 3) | cooking, cleaning, childcare, eldercare, marketing, pet_care, driving | ✓ |
| `daily_presence` | Select | all_day, morning_only, evening_only, mostly_empty | ✓ |
| `preferred_traits` | Multi-select (max 2) | patient, organized, proactive, quiet, outgoing, disciplined | ✓ |
| `special_care_needs` | Multi-select | infant, toddler, preschool, school_age, mobility_impaired_elderly, chronic_illness_elderly |  |
| `care_tasks` | Multi-select | feeding, bathing, medication, medical_escort, exercise, homework_help |  |
| `relationship_style` | Select | professional, family_like, balanced | ✓ |
| `household_rules` | Text | 自由填寫 |  |
| `preferred_start_date` | Date |  | ✓ |
| `experience_requirement` | Select | accept_newbie, similar_background, local_experience_required | ✓ |
| `service_type` | Select | matching_only, full_service | ✓ |
| `birthdate` | Date | 用於五行配對（選填） |  |

### 外傭註冊欄位（完整版）

| 欄位名稱 | 類型 | 選項／說明 | 必填 |
|---------|------|-----------|-----|
| `full_name` | Text |  | ✓ |
| `display_name` | Text | 對外顯示名 | ✓ |
| `nationality` | Select | Philippines, Indonesia, Sri Lanka, Myanmar, India, Other | ✓ |
| `birthdate` | Date |  | ✓ |
| `religion` | Select | Christian, Catholic, Muslim, Buddhist, Hindu, Other |  |
| `current_location` | Select | Hong Kong, Singapore, Overseas | ✓ |
| `contract_status` | Select | finishing, early_termination, transfer, freelance, overseas | ✓ |
| `available_from` | Date |  | ✓ |
| `years_experience_total` | Number |  | ✓ |
| `years_experience_local` | Number | HK/SG 經驗 | ✓ |
| `regions_worked` | Multi-select | Hong Kong, Singapore, Middle East, Taiwan, Other |  |
| `number_of_employers` | Number |  |  |
| `education_level` | Select | primary, secondary, tertiary | ✓ |
| `certifications` | Multi-select | first_aid, caregiving, cooking, driving |  |
| **技能** |  |  |  |
| `skill_cooking_chinese` | Select | none, basic, intermediate, advanced |  |
| `skill_cooking_western` | Select | none, basic, intermediate, advanced |  |
| `skill_cooking_local` | Select | (Indonesian, Filipino, etc.) |  |
| `skill_cleaning` | Select | none, basic, intermediate, advanced | ✓ |
| `skill_laundry_ironing` | Select | none, basic, intermediate, advanced |  |
| `skill_marketing` | Select | none, basic, intermediate, advanced |  |
| `skill_gardening` | Select | none, basic, intermediate, advanced |  |
| `skill_driving` | Select | none, basic, intermediate, advanced |  |
| **照顧經驗** |  |  |  |
| `care_infant` | Number | 年數 |  |
| `care_toddler` | Number |  |  |
| `care_child` | Number |  |  |
| `care_teenager` | Number |  |  |
| `care_healthy_elderly` | Number |  |  |
| `care_disabled_elderly` | Number |  |  |
| **語言** |  |  |  |
| `lang_english` | Select | none, basic, intermediate, fluent | ✓ |
| `lang_cantonese` | Select | none, basic, intermediate, fluent |  |
| `lang_mandarin` | Select | none, basic, intermediate, fluent |  |
| **性格與偏好** |  |  |  |
| `preferred_care_target` | Multi-select (rank) | infant, child, elderly, pets |  |
| `rest_day_flexibility` | Select | strict_sunday, weekday_ok, flexible |  |
| `accommodation_acceptance` | Select | own_room_required, can_share_with_child, flexible |  |
| `personality_traits` | Multi-select (max 3) | patient, organized, cheerful, quiet, proactive, disciplined | ✓ |
| `work_style` | Select | follow_instructions, self_arrange, balanced |  |
| `expected_salary_min` | Number |  |  |
| `expected_salary_max` | Number |  |  |
| `cannot_accept` | Multi-select | large_pets, male_elderly_bathing, heavy_lifting, etc. |  |
| `about_me` | Textarea | 自我介紹 |  |
| `profile_photo` | File Upload |  |  |

---

## 附錄：五行與星座計算邏輯

### 西洋星座計算

根據出生日期（月日）判斷星座：

```javascript
function getWesternZodiac(month, day) {
  const zodiacDates = [
    { name: 'capricorn', start: [12, 22], end: [1, 19] },
    { name: 'aquarius', start: [1, 20], end: [2, 18] },
    { name: 'pisces', start: [2, 19], end: [3, 20] },
    { name: 'aries', start: [3, 21], end: [4, 19] },
    { name: 'taurus', start: [4, 20], end: [5, 20] },
    { name: 'gemini', start: [5, 21], end: [6, 20] },
    { name: 'cancer', start: [6, 21], end: [7, 22] },
    { name: 'leo', start: [7, 23], end: [8, 22] },
    { name: 'virgo', start: [8, 23], end: [9, 22] },
    { name: 'libra', start: [9, 23], end: [10, 22] },
    { name: 'scorpio', start: [10, 23], end: [11, 21] },
    { name: 'sagittarius', start: [11, 22], end: [12, 21] }
  ];
  
  for (let zodiac of zodiacDates) {
    const [startMonth, startDay] = zodiac.start;
    const [endMonth, endDay] = zodiac.end;
    
    if (
      (month === startMonth && day >= startDay) ||
      (month === endMonth && day <= endDay)
    ) {
      return zodiac.name;
    }
  }
}
```

### 五行計算（簡化版）

完整八字需要出生年月日時的天干地支計算，MVP 階段可用簡化版：

**方法 1：根據出生年尾數（天干）**
```javascript
function getWuxingByYear(year) {
  const lastDigit = year % 10;
  const mapping = {
    0: 'metal', 1: 'metal',  // 庚辛
    2: 'water', 3: 'water',  // 壬癸
    4: 'wood',  5: 'wood',   // 甲乙
    6: 'fire',  7: 'fire',   // 丙丁
    8: 'earth', 9: 'earth'   // 戊己
  };
  return mapping[lastDigit];
}
```

**方法 2：根據星座元素**
```javascript
function getWuxingByZodiac(zodiac) {
  const zodiacToElement = {
    'aries': 'fire', 'leo': 'fire', 'sagittarius': 'fire',
    'taurus': 'earth', 'virgo': 'earth', 'capricorn': 'earth',
    'gemini': 'wood', 'libra': 'wood', 'aquarius': 'wood', // Air → Wood (變通)
    'cancer': 'water', 'scorpio': 'water', 'pisces': 'water'
  };
  return zodiacToElement[zodiac];
}
```

MVP 階段建議使用方法 1（年份）或方法 2（星座映射），待日後有資源再實作完整八字計算。

### 五行相生相剋判斷

```javascript
function getElementRelation(element1, element2) {
  const generate = {
    'water': 'wood',
    'wood': 'fire',
    'fire': 'earth',
    'earth': 'metal',
    'metal': 'water'
  };
  
  const overcome = {
    'water': 'fire',
    'fire': 'metal',
    'metal': 'wood',
    'wood': 'earth',
    'earth': 'water'
  };
  
  if (element1 === element2) return 'same';
  if (generate[element1] === element2) return 'generate'; // 相生
  if (overcome[element1] === element2) return 'overcome'; // 相剋
  return 'neutral';
}

function calculateElementBonus(employerElement, helperElement) {
  const relation = getElementRelation(employerElement, helperElement);
  
  switch(relation) {
    case 'generate': return 5;
    case 'same': return 3;
    case 'overcome': return -2;
    default: return 0;
  }
}
```

---

## 結語

這份規格書涵蓋了 Peasy / HelperMate 平台從概念到實作的完整藍圖：

✅ 市場定位與競品差異分析  
✅ 完整技術架構與資料模型  
✅ 詳細功能模組設計  
✅ 配對邏輯（含五行星座）  
✅ HSBC 風格 Design System  
✅ 開發階段與時程規劃  
✅ API 端點與表單欄位清單

**下一步行動**:
1. 確認技術堆疊選擇（Next.js / NestJS 或其他）
2. 設置開發環境與 repo
3. 實作 Phase 1 MVP（認證 + 問卷 + 註冊）
4. 迭代優化配對算法
5. 逐步推出付費功能與進階服務

這份文件可直接交給 AI agent、工程師、設計師使用，讓整個團隊對齊願景與執行細節。

---

**文件維護**: 隨開發進度更新版本號與修改日期  
**聯絡**: [你的聯絡方式]  
**授權**: 內部使用