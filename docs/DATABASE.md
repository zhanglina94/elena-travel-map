# 艾琳娜的漫游地图｜数据库设计文档（DATABASE）

- 版本：V1.0
- 数据库：PostgreSQL
- ORM：Prisma
- 状态：开发基线，待评审

## 1. 设计原则
1. 国家、城市、路线、节点、交通、旅行经验、签证经历和商品分表。
2. 关键事实保留来源、发生日期和核验状态。
3. 路线节点允许同一城市重复出现。
4. 历史价格与当前报价必须区分。
5. 个人签证经历与当前政策分离。
6. 商品文件私有存储，购买后由 API 鉴权并签发短时下载 URL。
7. 发布、下架和历史版本可追溯。
8. 金额使用 `NUMERIC/Decimal`，不使用浮点数。
9. 时间统一使用 `timestamptz`，按用户时区展示。

## 2. ER 关系
- countries 1:N cities
- routes 1:N route_stops
- cities 1:N route_stops
- route_stops 1:N transport_records（起点/终点分别关联）
- routes 1:N travel_notes
- countries 1:N visa_records
- products 1:N product_files
- routes N:N products（route_products）
- users 1:N orders；orders 1:N order_items
- users N:N routes（collections）
- users 1:N itineraries；itineraries 1:N itinerary_days

## 3. PostgreSQL DDL

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TYPE route_type AS ENUM ('loop','one_way','multi_country');
CREATE TYPE content_status AS ENUM ('draft','review','published','archived');
CREATE TYPE verification_status AS ENUM ('unverified','partial','verified');
CREATE TYPE note_type AS ENUM ('story','tip','transport','safety','food');
CREATE TYPE source_type AS ENUM ('personal_record','official','other');
CREATE TYPE product_status AS ENUM ('draft','published','archived');
CREATE TYPE order_status AS ENUM ('pending','paid','closed','refunded','failed');
CREATE TYPE delivery_type AS ENUM ('digital_file','external_service');

CREATE TABLE countries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name_zh VARCHAR(120) NOT NULL,
  name_en VARCHAR(120) NOT NULL,
  iso2 CHAR(2) UNIQUE,
  region VARCHAR(80),
  cover_url TEXT,
  description TEXT,
  is_published BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE cities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  country_id UUID NOT NULL REFERENCES countries(id),
  name_zh VARCHAR(120) NOT NULL,
  name_en VARCHAR(120),
  latitude NUMERIC(9,6),
  longitude NUMERIC(9,6),
  cover_url TEXT,
  description TEXT,
  is_published BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(country_id, name_en)
);

CREATE TABLE routes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug VARCHAR(180) NOT NULL UNIQUE,
  title VARCHAR(200) NOT NULL,
  subtitle VARCHAR(300),
  route_type route_type NOT NULL,
  region VARCHAR(100),
  duration_min SMALLINT,
  duration_max SMALLINT,
  difficulty VARCHAR(40),
  theme_tags TEXT[] NOT NULL DEFAULT '{}',
  content_grade CHAR(1) NOT NULL DEFAULT 'C' CHECK (content_grade IN ('A','B','C')),
  description TEXT,
  cover_url TEXT,
  content_status content_status NOT NULL DEFAULT 'draft',
  verification_status verification_status NOT NULL DEFAULT 'unverified',
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (duration_min IS NULL OR duration_min > 0),
  CHECK (duration_max IS NULL OR duration_max > 0),
  CHECK (duration_min IS NULL OR duration_max IS NULL OR duration_max >= duration_min)
);

CREATE TABLE route_stops (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  route_id UUID NOT NULL REFERENCES routes(id) ON DELETE CASCADE,
  city_id UUID NOT NULL REFERENCES cities(id),
  sequence INTEGER NOT NULL,
  suggested_days NUMERIC(4,1),
  actual_days NUMERIC(4,1),
  arrival_note TEXT,
  experience_note TEXT,
  is_confirmed BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(route_id, sequence),
  CHECK (sequence > 0),
  CHECK (suggested_days IS NULL OR suggested_days >= 0),
  CHECK (actual_days IS NULL OR actual_days >= 0)
);
-- 不要添加 UNIQUE(route_id, city_id)：同一城市可能在路线中重复出现。

CREATE TABLE transport_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  from_stop_id UUID NOT NULL REFERENCES route_stops(id) ON DELETE CASCADE,
  to_stop_id UUID NOT NULL REFERENCES route_stops(id) ON DELETE CASCADE,
  transport_mode VARCHAR(60) NOT NULL,
  operator VARCHAR(160),
  duration_minutes INTEGER,
  price_amount NUMERIC(12,2),
  currency CHAR(3),
  price_context VARCHAR(30) NOT NULL DEFAULT 'historical'
    CHECK (price_context IN ('historical','current_quote','unknown')),
  travel_date DATE,
  booking_url TEXT,
  source_type source_type NOT NULL DEFAULT 'personal_record',
  source_url TEXT,
  verified_at TIMESTAMPTZ,
  note TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (from_stop_id <> to_stop_id),
  CHECK (duration_minutes IS NULL OR duration_minutes >= 0),
  CHECK (price_amount IS NULL OR price_amount >= 0)
);

CREATE TABLE travel_notes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  route_id UUID REFERENCES routes(id) ON DELETE SET NULL,
  city_id UUID REFERENCES cities(id) ON DELETE SET NULL,
  title VARCHAR(200) NOT NULL,
  note_type note_type NOT NULL,
  content TEXT NOT NULL,
  travel_date DATE,
  source_type source_type NOT NULL DEFAULT 'personal_record',
  source_url TEXT,
  verification_status verification_status NOT NULL DEFAULT 'unverified',
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (route_id IS NOT NULL OR city_id IS NOT NULL)
);

CREATE TABLE visa_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  country_id UUID NOT NULL REFERENCES countries(id),
  nationality VARCHAR(100) NOT NULL,
  visa_type VARCHAR(100),
  experience_date DATE,
  entry_method VARCHAR(160),
  actual_experience TEXT NOT NULL,
  official_source_url TEXT,
  verified_at TIMESTAMPTZ,
  disclaimer TEXT NOT NULL DEFAULT '个人历史经历不代表当前签证政策。',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(200) NOT NULL,
  slug VARCHAR(180) NOT NULL UNIQUE,
  description TEXT,
  price NUMERIC(10,2) NOT NULL,
  currency CHAR(3) NOT NULL DEFAULT 'CNY',
  product_type VARCHAR(40) NOT NULL,
  cover_url TEXT,
  status product_status NOT NULL DEFAULT 'draft',
  delivery_type delivery_type NOT NULL DEFAULT 'digital_file',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (price >= 0)
);

CREATE TABLE product_files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  file_name VARCHAR(255) NOT NULL,
  storage_key TEXT NOT NULL,
  file_type VARCHAR(80) NOT NULL,
  version VARCHAR(40) NOT NULL DEFAULT '1.0',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(product_id, storage_key)
);

CREATE TABLE route_products (
  route_id UUID NOT NULL REFERENCES routes(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  sort_order INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY(route_id, product_id)
);

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wx_openid VARCHAR(160) UNIQUE NOT NULL,
  wx_unionid VARCHAR(160),
  nickname VARCHAR(120),
  avatar_url TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE collections (
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  route_id UUID NOT NULL REFERENCES routes(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY(user_id, route_id)
);

CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_no VARCHAR(64) NOT NULL UNIQUE,
  user_id UUID NOT NULL REFERENCES users(id),
  status order_status NOT NULL DEFAULT 'pending',
  amount NUMERIC(10,2) NOT NULL,
  currency CHAR(3) NOT NULL DEFAULT 'CNY',
  wx_transaction_id VARCHAR(128) UNIQUE,
  paid_at TIMESTAMPTZ,
  closed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (amount >= 0)
);

CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  product_title_snapshot VARCHAR(200) NOT NULL,
  unit_price_snapshot NUMERIC(10,2) NOT NULL,
  quantity INTEGER NOT NULL DEFAULT 1,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (unit_price_snapshot >= 0),
  CHECK (quantity > 0)
);

CREATE TABLE itineraries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(200) NOT NULL,
  start_date DATE,
  end_date DATE,
  status VARCHAR(30) NOT NULL DEFAULT 'draft',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (start_date IS NULL OR end_date IS NULL OR end_date >= start_date)
);

CREATE TABLE itinerary_days (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  itinerary_id UUID NOT NULL REFERENCES itineraries(id) ON DELETE CASCADE,
  day_no INTEGER NOT NULL,
  travel_date DATE,
  city_id UUID REFERENCES cities(id),
  title VARCHAR(200),
  note TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(itinerary_id, day_no),
  CHECK (day_no > 0)
);

CREATE TABLE itinerary_budget_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  itinerary_id UUID NOT NULL REFERENCES itineraries(id) ON DELETE CASCADE,
  itinerary_day_id UUID REFERENCES itinerary_days(id) ON DELETE SET NULL,
  category VARCHAR(80) NOT NULL,
  amount NUMERIC(12,2) NOT NULL,
  currency CHAR(3) NOT NULL,
  note TEXT,
  spent_on DATE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (amount >= 0)
);

CREATE TABLE admin_users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  role VARCHAR(40) NOT NULL DEFAULT 'editor',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  admin_user_id UUID REFERENCES admin_users(id) ON DELETE SET NULL,
  action VARCHAR(100) NOT NULL,
  entity_type VARCHAR(80) NOT NULL,
  entity_id UUID,
  before_data JSONB,
  after_data JSONB,
  ip_address INET,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE route_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  route_id UUID NOT NULL REFERENCES routes(id) ON DELETE CASCADE,
  version_no INTEGER NOT NULL,
  snapshot JSONB NOT NULL,
  created_by UUID REFERENCES admin_users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(route_id, version_no)
);

CREATE INDEX idx_cities_country ON cities(country_id);
CREATE INDEX idx_routes_public ON routes(content_status, region, published_at DESC);
CREATE INDEX idx_routes_type ON routes(route_type);
CREATE INDEX idx_route_stops_route_sequence ON route_stops(route_id, sequence);
CREATE INDEX idx_transport_from_stop ON transport_records(from_stop_id);
CREATE INDEX idx_transport_to_stop ON transport_records(to_stop_id);
CREATE INDEX idx_travel_notes_route ON travel_notes(route_id, published_at DESC);
CREATE INDEX idx_visa_records_country ON visa_records(country_id, experience_date DESC);
CREATE INDEX idx_products_public ON products(status, created_at DESC);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_itineraries_user ON itineraries(user_id, updated_at DESC);
CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id, created_at DESC);
```

## 4. 字段语义与约束

### routes
- `route_type`：`loop` 闭合环线；`one_way` 单程；`multi_country` 多国路线。
- `content_grade`：A 完整亲历；B 部分亲历；C 旅行灵感。
- `verification_status`：未核验/部分核验/已核验。
- `content_status`：草稿/审核中/已发布/已归档。

### transport_records
- `historical`：历史发生价格，配合 `travel_date` 展示。
- `current_quote`：当前报价，必须保存来源和核验时间。
- `unknown`：未知价格，不得用 0 表示未知。
- 来源区分个人记录、官方来源和其他来源。

### visa_records
该表只保存个人经历。若未来管理现行政策，应另建政策表，包含适用国籍、生效时间、官方来源和核验时间，不得覆盖历史经历。

### 文件交付
`product_files.storage_key` 保存对象存储 key，不保存公开永久 URL。下载 API 必须校验登录用户、订单归属、订单已支付、订单项包含商品及文件 active 状态，再生成短时签名 URL。

## 5. Prisma 建模提醒
- UUID 使用 `@db.Uuid`。
- 金额使用 `Decimal @db.Decimal(10, 2)`。
- PostgreSQL enum 在 Prisma 中定义为 `enum`。
- `theme_tags` 可使用 `String[]`。
- 多对多关系建议显式建模 `RouteProduct`。
- `route_stops` 只对 `(routeId, sequence)` 唯一，不对 `(routeId, cityId)` 唯一。
- 生产结构变更只通过 Prisma Migrate 管理。

## 6. 迁移与 seed
1. 建立空数据库。
2. 创建 Prisma schema。
3. `prisma migrate dev --name init`。
4. 部署环境使用 `prisma migrate deploy`。
5. seed 仅导入已确认的国家和城市基础数据。
6. 路线及交通内容先作为草稿导入，人工核验后发布。
7. 不自动补全未经确认的城市、日期、交通或价格。

## 7. 上线前检查
- [ ] 外键和删除策略符合业务预期。
- [ ] 节点 sequence 唯一且经检查。
- [ ] 同一路线允许重复城市。
- [ ] 金额使用 Decimal。
- [ ] 订单号、微信交易号唯一。
- [ ] 支付回调具备幂等处理。
- [ ] 私有文件不向未授权用户暴露。
- [ ] 个人签证经历带免责声明。
- [ ] 历史交通价格带发生日期。
- [ ] 发布状态和核验状态分离。
