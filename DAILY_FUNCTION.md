# Daily.co 語音通話 - Edge Function

## 函數名稱：`daily-room`

在 Supabase Dashboard 建立這個 Edge Function：

```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};

serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  try {
    const DAILY_API_KEY = Deno.env.get("DAILY_API_KEY");
    if (!DAILY_API_KEY) {
      return new Response(JSON.stringify({ error: "Daily API key not configured" }), {
        status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    const { roomName, userName } = await req.json();
    
    if (!roomName) {
      return new Response(JSON.stringify({ error: "Room name required" }), {
        status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    // 檢查房間是否存在，不存在就建立
    const dailyRoomName = `meeting-${roomName}`;
    
    let roomUrl = "";
    
    // 嘗試取得現有房間
    const getRes = await fetch(`https://api.daily.co/v1/rooms/${dailyRoomName}`, {
      headers: { "Authorization": `Bearer ${DAILY_API_KEY}` }
    });

    if (getRes.ok) {
      const room = await getRes.json();
      roomUrl = room.url;
    } else {
      // 建立新房間
      const createRes = await fetch("https://api.daily.co/v1/rooms", {
        method: "POST",
        headers: {
          "Authorization": `Bearer ${DAILY_API_KEY}`,
          "Content-Type": "application/json"
        },
        body: JSON.stringify({
          name: dailyRoomName,
          properties: {
            enable_chat: false,
            enable_screenshare: false,
            start_video_off: true,
            start_audio_off: false,
            exp: Math.floor(Date.now() / 1000) + 86400 // 24 小時後過期
          }
        })
      });

      if (!createRes.ok) {
        const err = await createRes.json();
        throw new Error(err.error || "Failed to create room");
      }

      const room = await createRes.json();
      roomUrl = room.url;
    }

    // 建立 meeting token（讓使用者加入）
    const tokenRes = await fetch("https://api.daily.co/v1/meeting-tokens", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${DAILY_API_KEY}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        properties: {
          room_name: dailyRoomName,
          user_name: userName || "Guest",
          exp: Math.floor(Date.now() / 1000) + 3600 // 1 小時後過期
        }
      })
    });

    if (!tokenRes.ok) {
      const err = await tokenRes.json();
      throw new Error(err.error || "Failed to create token");
    }

    const { token } = await tokenRes.json();

    return new Response(JSON.stringify({ 
      roomUrl, 
      token,
      success: true 
    }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" }
    });

  } catch (e) {
    console.error("Error:", e);
    return new Response(JSON.stringify({ error: String(e) }), {
      status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" }
    });
  }
});
```

---

## 部署步驟

1. 在 Supabase Dashboard → Edge Functions → 建立 `daily-room`
2. 貼上上面的程式碼
3. 關閉 JWT Verification
4. 在 Supabase → Settings → Secrets 新增：
   - Name: `DAILY_API_KEY`
   - Value: 你的 Daily.co API Key
