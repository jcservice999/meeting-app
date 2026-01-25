# Supabase 資料庫設定 - 房間權限控制

## 步驟 1：創建 rooms 表格

在 Supabase SQL Editor 執行：

```sql
-- 創建 rooms 表格
CREATE TABLE rooms (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  room_id TEXT UNIQUE NOT NULL,
  room_name TEXT NOT NULL,
  created_by UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  is_active BOOLEAN DEFAULT TRUE,
  session_start_time TEXT  -- 🟢 新增：用於防止重新整理頁面後紀錄遺失
);

-- 啟用 RLS
ALTER TABLE rooms ENABLE ROW LEVEL SECURITY;

-- 政策：所有人可以讀取活躍的房間
CREATE POLICY "Anyone can view active rooms"
  ON rooms FOR SELECT
  USING (is_active = true);

-- 政策：已登入使用者可以創建房間
CREATE POLICY "Authenticated users can create rooms"
  ON rooms FOR INSERT
  WITH CHECK (auth.uid() = created_by);

-- 政策：創建者可以更新自己的房間
CREATE POLICY "Users can update their own rooms"
  ON rooms FOR UPDATE
  USING (auth.uid() = created_by);
```

## 步驟 2：創建 room_access 表格

```sql
-- 創建 room_access 表格
CREATE TABLE room_access (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  room_id TEXT REFERENCES rooms(room_id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  added_by UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  added_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(room_id, email)
);

-- 啟用 RLS
ALTER TABLE room_access ENABLE ROW LEVEL SECURITY;

-- 政策：使用者可以查看自己有權限的房間
CREATE POLICY "Users can view their own access"
  ON room_access FOR SELECT
  USING (email = auth.jwt()->>'email');

-- 政策：房間創建者可以管理存取列表
CREATE POLICY "Room creators can manage access"
  ON room_access FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM rooms
      WHERE rooms.room_id = room_access.room_id
      AND rooms.created_by = auth.uid()
    )
  );
```

## 步驟 3：創建輔助函數

```sql
-- 檢查使用者是否有房間存取權限
CREATE OR REPLACE FUNCTION check_room_access(p_room_id TEXT, p_email TEXT)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM room_access
    WHERE room_id = p_room_id
    AND email = p_email
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## 完成後

執行完以上 SQL 後，您的資料庫就準備好了！
