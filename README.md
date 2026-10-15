-- ============================================
--  EMOTE MENU v16 — SPY + Categories + Library
-- ============================================

local Players           = game:GetService("Players")
local UserInputService  = game:GetService("UserInputService")
local RunService        = game:GetService("RunService")
local TweenService      = game:GetService("TweenService")
local CoreGui           = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local SoundService      = game:GetService("SoundService")
local HttpService       = game:GetService("HttpService")

local lp = Players.LocalPlayer

-- CLEANUP
if _G.EmoteMenuCleanup then pcall(_G.EmoteMenuCleanup) end
do
    local function kill(parent)
        pcall(function()
            for _, g in ipairs(parent:GetChildren()) do
                if g.Name == "EmoteMenuGui" then g:Destroy() end
            end
        end)
    end
    kill(CoreGui)
    local pg = lp:FindFirstChild("PlayerGui")
    if pg then kill(pg) end
end

-- SONS
local SOUNDS = {
    toggleOn   = "rbxassetid://134815520229822",
    toggleOff  = "rbxassetid://134815520229822",
    volume     = 0.5,
}

local function playSound(soundId)
    if not soundId or soundId == "" then return end
    pcall(function()
        local s = Instance.new("Sound")
        s.SoundId = soundId
        s.Volume = SOUNDS.volume or 0.5
        s.Parent = SoundService
        s:Play()
        s.Ended:Connect(function() s:Destroy() end)
        task.delay(5, function() pcall(function() s:Destroy() end) end)
    end)
end

-- LIBRARY
local LIBRARY_FILE = "mm2_emote_library.json"
local LIBRARY = {}
local libraryRows = {}

local function loadLibrary()
    if not isfile or not readfile then return end
    pcall(function()
        if isfile(LIBRARY_FILE) then
            local data = readfile(LIBRARY_FILE)
            local ok, decoded = pcall(function() return HttpService:JSONDecode(data) end)
            if ok and type(decoded) == "table" then
                LIBRARY = decoded
            end
        end
    end)
end

local function saveLibrary()
    if not writefile then return end
    pcall(function()
        writefile(LIBRARY_FILE, HttpService:JSONEncode(LIBRARY))
    end)
end

-- STATE
local State = {
    open = true,
    currentTab = "emotes",
    emoteTrack = nil,
    currentId = nil,
    pendingId = nil,
    autoEmote = false,
    chams = false,
    gunESP = false,
    autoGrab = false,
    speed = false,
    speedValue = 19,
    defaultSpeed = 16,
    roleCache = {},
    aimLock = false,
    aimMode = "SMOOTH",
    aimFOV = 200,
    aimSmoothness = 0.65,
    aimJitter = 0.008,
    aimUseKey = false,
    aimKey = Enum.KeyCode.E,
    aimCurrentTarget = nil,
    grabbing = false,
    lastGrabAttempt = 0,
    spyEnabled = false,
    spySeen = {},
    spyTimeout = 10,
    spyFilter = "ALL",   -- "ALL" | "IDLE" | "WALK" | "EMOTE" | "CUSTOM"
}
local Binds = {}
local BindWaiting = nil
local conns = {}
local HOLD_KEY = Enum.KeyCode.LeftAlt
local LOAD_AT = os.clock()
local chamsInstances = {}
local gunHighlights = {}
local lastChamsRefresh = 0
local lastRoleRefresh = 0
local lastSpyRefresh = 0

local function track(c) table.insert(conns, c); return c end

-- EMOTES CONNUES
local EMOTES = {
    {name="Zen",        id=2431812646, color=Color3.fromRGB(0,240,255)},
    {name="Ninja",      id=2431864798, color=Color3.fromRGB(176,38,255)},
    {name="Dab",        id=2445521505, color=Color3.fromRGB(255,140,60)},
    {name="Floss",      id=2452938820, color=Color3.fromRGB(57,255,136)},
    {name="Headless",   id=2513664073, color=Color3.fromRGB(255,60,80)},
    {name="Zombie",     id=2513692312, color=Color3.fromRGB(120,220,80)},
    {name="Sit",        id=2431845940, color=Color3.fromRGB(80,140,255)},
}

local EMOTE_DB = {}

local EXCLUDE_KEYWORDS = {
    "slash", "punch", "shoot", "attack", "hit", "swing",
    "stab", "kill", "damage", "sword", "knife", "gun",
    "throw", "block", "dodge", "roll", "parry", "reload",
    "equip", "unequip", "pull", "fire", "aim",
    "chop", "cut", "slice", "melee", "combat",
    "clash", "flick", "swipe",
}

local EXCLUDE_EXACT_NAMES = {
    "Idle", "Walk", "Run", "Jump", "Fall", "Climb", "Swim",
    "Sit", "Tool", "Handle", "Attack", "Slash", "Stab",
}

local function isExcludedName(name)
    if not name or name == "" then return false end
    for _, exact in ipairs(EXCLUDE_EXACT_NAMES) do
        if name == exact then return true end
    end
    local lower = name:lower()
    for _, kw in ipairs(EXCLUDE_KEYWORDS) do
        if lower:find(kw, 1, true) then return true end
    end
    return false
end

local function normalizeAnimId(id)
    if not id then return nil end
    id = id:gsub("rbxassetid://", "")
    id = id:gsub("http://www%.roblox%.com/asset/%?id=", "")
    id = id:gsub("https://www%.roblox%.com/asset/%?id=", "")
    return id
end

-- Catégorise une anim
local function categorizeAnim(prio, len)
    if prio == Enum.AnimationPriority.Idle then
        return "IDLE"
    end
    if prio == Enum.AnimationPriority.Movement then
        return "WALK"
    end
    if prio == Enum.AnimationPriority.Core then
        if len < 1.0 then
            return "WALK"
        else
            return "CUSTOM"
        end
    end
    if prio == Enum.AnimationPriority.Action
       or prio == Enum.AnimationPriority.Action2
       or prio == Enum.AnimationPriority.Action3
       or prio == Enum.AnimationPriority.Action4 then
        return "EMOTE"
    end
    return "CUSTOM"
end

-- Couleur par catégorie
local CAT_COLORS = {
    IDLE   = Color3.fromRGB(100, 160, 255),  -- bleu
    WALK   = Color3.fromRGB(120, 220, 80),   -- vert
    EMOTE  = Color3.fromRGB(255, 214, 0),    -- jaune
    CUSTOM = Color3.fromRGB(200, 140, 255),  -- violet
}

for _, e in ipairs(EMOTES) do
    EMOTE_DB["rbxassetid://" .. e.id] = e.name
    EMOTE_DB[tostring(e.id)] = e.name
end

local ROLE_COLORS = {
    Murderer = Color3.fromRGB(255, 60, 80),
    Sheriff  = Color3.fromRGB(60, 140, 255),
    Innocent = Color3.fromRGB(80, 220, 100),
    Hero     = Color3.fromRGB(255, 250, 0),
}

-- ROLE DETECTION
local function updateRoleCache()
    local ok, data = pcall(function()
        local getPlayerData = ReplicatedStorage:FindFirstChild("GetPlayerData", true)
        if getPlayerData and getPlayerData:IsA("RemoteFunction") then
            return getPlayerData:InvokeServer()
        end
        return nil
    end)
    if not ok or type(data) ~= "table" then return end
    State.roleCache = {}
    for playerName, playerData in pairs(data) do
        if type(playerData) == "table" and playerData.Role then
            State.roleCache[playerName] = playerData.Role
        end
    end
end

local function getPlayerRole(plr)
    return State.roleCache[plr.Name] or "Innocent"
end

-- PALETTE
local T = {
    bg=Color3.fromRGB(12,14,20), bg2=Color3.fromRGB(18,22,30),
    card=Color3.fromRGB(22,26,36), cardHover=Color3.fromRGB(32,38,50),
    text=Color3.fromRGB(245,248,255), subtext=Color3.fromRGB(140,150,175),
    dim=Color3.fromRGB(90,100,125), accent=Color3.fromRGB(120,200,255),
    accent2=Color3.fromRGB(200,140,255), green=Color3.fromRGB(80,240,160),
    red=Color3.fromRGB(255,90,110), yellow=Color3.fromRGB(255,215,80),
    gold=Color3.fromRGB(255,195,80), blue=Color3.fromRGB(100,160,255),
    stroke=Color3.fromRGB(45,52,70), strokeLight=Color3.fromRGB(60,70,90),
}

local function corner(p,r) local c=Instance.new("UICorner"); c.CornerRadius=UDim.new(0,r or 8); c.Parent=p; return c end
local function stroke(p,col,th,tr) local s=Instance.new("UIStroke"); s.Color=col or T.stroke; s.Thickness=th or 1; s.Transparency=tr or 0; s.Parent=p; return s end
local function tween(o,t,props,st,dir) local tw=TweenService:Create(o,TweenInfo.new(t,st or Enum.EasingStyle.Quad,dir or Enum.EasingDirection.Out),props); tw:Play(); return tw end
local function gradient(p,c1,c2,rot) local g=Instance.new("UIGradient"); g.Color=ColorSequence.new(c1,c2); g.Rotation=rot or 0; g.Parent=p; return g end
local function gradient3(p,c1,c2,c3,rot)
    local g=Instance.new("UIGradient")
    g.Color=ColorSequence.new({ColorSequenceKeypoint.new(0,c1),ColorSequenceKeypoint.new(0.5,c2),ColorSequenceKeypoint.new(1,c3)})
    g.Rotation=rot or 0; g.Parent=p; return g
end

-- GUI
local gui = Instance.new("ScreenGui")
gui.Name = "EmoteMenuGui"
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.IgnoreGuiInset = true
pcall(function() gui.Parent = CoreGui end)
if not gui.Parent then gui.Parent = lp:WaitForChild("PlayerGui") end

local panel = Instance.new("Frame")
panel.Size = UDim2.new(0, 520, 0, 600)
panel.Position = UDim2.new(0, 30, 0.5, -300)
panel.BackgroundColor3 = T.bg
panel.BackgroundTransparency = 0.02
panel.BorderSizePixel = 0
panel.ClipsDescendants = true
panel.Parent = gui
corner(panel, 16)
gradient(panel, T.bg, T.bg2, 90)
stroke(panel, T.strokeLight, 1, 0.4)

local topGlow = Instance.new("Frame")
topGlow.Size = UDim2.new(1, 0, 0, 80)
topGlow.BackgroundColor3 = T.accent
topGlow.BackgroundTransparency = 0.9
topGlow.BorderSizePixel = 0
topGlow.Parent = panel
corner(topGlow, 16)
gradient3(topGlow, T.accent, T.accent2, T.bg, 90)

-- HEADER
local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 50)
header.BackgroundTransparency = 1
header.BorderSizePixel = 0
header.Parent = panel
header.Active = true

local logoBox = Instance.new("Frame")
logoBox.Size = UDim2.new(0, 32, 0, 32)
logoBox.Position = UDim2.new(0, 14, 0.5, -16)
logoBox.BackgroundColor3 = T.accent
logoBox.BorderSizePixel = 0
logoBox.Parent = header
corner(logoBox, 9)
gradient(logoBox, T.accent, T.accent2, 45)

local logoText = Instance.new("TextLabel")
logoText.Size = UDim2.new(1, 0, 1, 0)
logoText.BackgroundTransparency = 1
logoText.Text = "M"
logoText.TextColor3 = Color3.new(0,0,0)
logoText.Font = Enum.Font.GothamBlack
logoText.TextSize = 18
logoText.Parent = logoBox

local title = Instance.new("TextLabel")
title.Size = UDim2.new(0, 200, 0, 18)
title.Position = UDim2.new(0, 56, 0, 8)
title.BackgroundTransparency = 1
title.Text = "EMOTE MENU"
title.TextColor3 = T.text
title.Font = Enum.Font.GothamBlack
title.TextSize = 14
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

local subtitle = Instance.new("TextLabel")
subtitle.Size = UDim2.new(0, 200, 0, 12)
subtitle.Position = UDim2.new(0, 56, 0, 26)
subtitle.BackgroundTransparency = 1
subtitle.Text = "v16 • CATEGORIES"
subtitle.TextColor3 = T.accent
subtitle.Font = Enum.Font.Code
subtitle.TextSize = 9
subtitle.TextXAlignment = Enum.TextXAlignment.Left
subtitle.Parent = header

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 26, 0, 26)
closeBtn.Position = UDim2.new(1, -34, 0.5, -13)
closeBtn.BackgroundColor3 = T.card
closeBtn.TextColor3 = T.subtext
closeBtn.Text = "✕"
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 12
closeBtn.AutoButtonColor = false
closeBtn.Parent = header
corner(closeBtn, 8)
stroke(closeBtn, T.stroke, 1, 0.5)

local sep = Instance.new("Frame")
sep.Size = UDim2.new(1, -28, 0, 1)
sep.Position = UDim2.new(0, 14, 0, 58)
sep.BackgroundColor3 = T.stroke
sep.BorderSizePixel = 0
sep.Parent = panel
gradient3(sep, T.bg, T.stroke, T.bg, 0)

-- TABS
local tabBar = Instance.new("Frame")
tabBar.Size = UDim2.new(1, -28, 0, 34)
tabBar.Position = UDim2.new(0, 14, 0, 68)
tabBar.BackgroundColor3 = T.card
tabBar.BackgroundTransparency = 0.3
tabBar.BorderSizePixel = 0
tabBar.Parent = panel
corner(tabBar, 10)
stroke(tabBar, T.stroke, 1, 0.5)

local emotesTab = Instance.new("TextButton")
emotesTab.Size = UDim2.new(0.25, -2, 1, -6)
emotesTab.Position = UDim2.new(0, 3, 0, 3)
emotesTab.BackgroundColor3 = T.accent
emotesTab.TextColor3 = Color3.fromRGB(10,14,20)
emotesTab.Font = Enum.Font.GothamBold
emotesTab.TextSize = 10
emotesTab.Text = "EMOTES"
emotesTab.AutoButtonColor = false
emotesTab.Parent = tabBar
corner(emotesTab, 7)
gradient(emotesTab, T.accent, T.accent2, 0)

local libraryTab = Instance.new("TextButton")
libraryTab.Size = UDim2.new(0.25, -2, 1, -6)
libraryTab.Position = UDim2.new(0.25, 0, 0, 3)
libraryTab.BackgroundColor3 = T.card
libraryTab.BackgroundTransparency = 1
libraryTab.TextColor3 = T.subtext
libraryTab.Font = Enum.Font.GothamBold
libraryTab.TextSize = 10
libraryTab.Text = "LIBRARY"
libraryTab.AutoButtonColor = false
libraryTab.Parent = tabBar
corner(libraryTab, 7)

local spyTab = Instance.new("TextButton")
spyTab.Size = UDim2.new(0.25, -2, 1, -6)
spyTab.Position = UDim2.new(0.50, 0, 0, 3)
spyTab.BackgroundColor3 = T.card
spyTab.BackgroundTransparency = 1
spyTab.TextColor3 = T.subtext
spyTab.Font = Enum.Font.GothamBold
spyTab.TextSize = 10
spyTab.Text = "SPY"
spyTab.AutoButtonColor = false
spyTab.Parent = tabBar
corner(spyTab, 7)

local utilsTab = Instance.new("TextButton")
utilsTab.Size = UDim2.new(0.25, -2, 1, -6)
utilsTab.Position = UDim2.new(0.75, -3, 0, 3)
utilsTab.BackgroundColor3 = T.card
utilsTab.BackgroundTransparency = 1
utilsTab.TextColor3 = T.subtext
utilsTab.Font = Enum.Font.GothamBold
utilsTab.TextSize = 10
utilsTab.Text = "UTILS"
utilsTab.AutoButtonColor = false
utilsTab.Parent = tabBar
corner(utilsTab, 7)

local content = Instance.new("Frame")
content.Size = UDim2.new(1, -28, 1, -158)
content.Position = UDim2.new(0, 14, 0, 112)
content.BackgroundTransparency = 1
content.Parent = panel

-- PAGE EMOTES
local emotesPage = Instance.new("ScrollingFrame")
emotesPage.Size = UDim2.new(1, 0, 1, 0)
emotesPage.BackgroundTransparency = 1
emotesPage.BorderSizePixel = 0
emotesPage.ScrollBarThickness = 3
emotesPage.ScrollBarImageColor3 = T.accent
emotesPage.ScrollBarImageTransparency = 0.5
emotesPage.CanvasSize = UDim2.new(0, 0, 0, 0)
emotesPage.AutomaticCanvasSize = Enum.AutomaticSize.Y
emotesPage.Parent = content

local emotesGrid = Instance.new("UIGridLayout")
emotesGrid.CellSize = UDim2.new(0, 106, 0, 62)
emotesGrid.CellPadding = UDim2.new(0, 8, 0, 8)
emotesGrid.SortOrder = Enum.SortOrder.LayoutOrder
emotesGrid.Parent = emotesPage

local emotesPad = Instance.new("UIPadding")
emotesPad.PaddingTop = UDim.new(0, 4)
emotesPad.PaddingBottom = UDim.new(0, 4)
emotesPad.PaddingLeft = UDim.new(0, 4)
emotesPad.PaddingRight = UDim.new(0, 8)
emotesPad.Parent = emotesPage

-- PAGE LIBRARY
local libraryPage = Instance.new("ScrollingFrame")
libraryPage.Size = UDim2.new(1, 0, 1, 0)
libraryPage.BackgroundTransparency = 1
libraryPage.BorderSizePixel = 0
libraryPage.ScrollBarThickness = 3
libraryPage.ScrollBarImageColor3 = T.accent
libraryPage.ScrollBarImageTransparency = 0.5
libraryPage.CanvasSize = UDim2.new(0, 0, 0, 0)
libraryPage.AutomaticCanvasSize = Enum.AutomaticSize.Y
libraryPage.Parent = content
libraryPage.Visible = false

local libraryGrid = Instance.new("UIGridLayout")
libraryGrid.CellSize = UDim2.new(0, 106, 0, 62)
libraryGrid.CellPadding = UDim2.new(0, 8, 0, 8)
libraryGrid.SortOrder = Enum.SortOrder.LayoutOrder
libraryGrid.Parent = libraryPage

local libraryPad = Instance.new("UIPadding")
libraryPad.PaddingTop = UDim.new(0, 4)
libraryPad.PaddingBottom = UDim.new(0, 4)
libraryPad.PaddingLeft = UDim.new(0, 4)
libraryPad.PaddingRight = UDim.new(0, 8)
libraryPad.Parent = libraryPage

local libraryEmptyLbl = Instance.new("TextLabel")
libraryEmptyLbl.Size = UDim2.new(1, -20, 0, 40)
libraryEmptyLbl.Position = UDim2.new(0, 10, 0.5, -20)
libraryEmptyLbl.BackgroundTransparency = 1
libraryEmptyLbl.Text = "Bibliothèque vide.\nUtilise le bouton [+] dans le SPY pour sauvegarder une emote."
libraryEmptyLbl.TextColor3 = T.dim
libraryEmptyLbl.Font = Enum.Font.Gotham
libraryEmptyLbl.TextSize = 10
libraryEmptyLbl.TextWrapped = true
libraryEmptyLbl.Parent = libraryPage

-- PAGE SPY
local spyPage = Instance.new("Frame")
spyPage.Size = UDim2.new(1, 0, 1, 0)
spyPage.BackgroundTransparency = 1
spyPage.Parent = content
spyPage.Visible = false

-- Input custom
local customFrame = Instance.new("Frame")
customFrame.Size = UDim2.new(1, 0, 0, 60)
customFrame.BackgroundColor3 = T.card
customFrame.BackgroundTransparency = 0.2
customFrame.BorderSizePixel = 0
customFrame.Parent = spyPage
corner(customFrame, 8)
stroke(customFrame, T.stroke, 1, 0.5)

local customLbl = Instance.new("TextLabel")
customLbl.Size = UDim2.new(1, -20, 0, 16)
customLbl.Position = UDim2.new(0, 10, 0, 6)
customLbl.BackgroundTransparency = 1
customLbl.Text = "CUSTOM EMOTE — Colle un ID"
customLbl.TextColor3 = T.subtext
customLbl.Font = Enum.Font.GothamBold
customLbl.TextSize = 9
customLbl.TextXAlignment = Enum.TextXAlignment.Left
customLbl.Parent = customFrame

local customInput = Instance.new("TextBox")
customInput.Size = UDim2.new(1, -80, 0, 28)
customInput.Position = UDim2.new(0, 10, 0, 24)
customInput.BackgroundColor3 = T.bg
customInput.Text = ""
customInput.PlaceholderText = "rbxassetid://..."
customInput.PlaceholderColor3 = T.dim
customInput.TextColor3 = T.text
customInput.Font = Enum.Font.Code
customInput.TextSize = 11
customInput.TextXAlignment = Enum.TextXAlignment.Left
customInput.ClearTextOnFocus = false
customInput.Parent = customFrame
corner(customInput, 6)
stroke(customInput, T.stroke, 1, 0.5)

local customPlayBtn = Instance.new("TextButton")
customPlayBtn.Size = UDim2.new(0, 60, 0, 28)
customPlayBtn.Position = UDim2.new(1, -70, 0, 24)
customPlayBtn.BackgroundColor3 = T.accent
customPlayBtn.TextColor3 = Color3.fromRGB(10,14,20)
customPlayBtn.Font = Enum.Font.GothamBold
customPlayBtn.TextSize = 10
customPlayBtn.Text = "▶ JOUER"
customPlayBtn.AutoButtonColor = false
customPlayBtn.Parent = customFrame
corner(customPlayBtn, 6)
gradient(customPlayBtn, T.accent, T.accent2, 0)

-- Toggle SPY
local spyToggleBtn = Instance.new("TextButton")
spyToggleBtn.Size = UDim2.new(1, 0, 0, 32)
spyToggleBtn.Position = UDim2.new(0, 0, 0, 68)
spyToggleBtn.BackgroundColor3 = T.card
spyToggleBtn.TextColor3 = T.subtext
spyToggleBtn.Font = Enum.Font.GothamBold
spyToggleBtn.TextSize = 11
spyToggleBtn.Text = "SPY : OFF"
spyToggleBtn.AutoButtonColor = false
spyToggleBtn.Parent = spyPage
corner(spyToggleBtn, 8)
stroke(spyToggleBtn, T.stroke, 1, 0.5)

-- Filtres catégories
local filterBar = Instance.new("Frame")
filterBar.Size = UDim2.new(1, 0, 0, 28)
filterBar.Position = UDim2.new(0, 0, 0, 108)
filterBar.BackgroundTransparency = 1
filterBar.Parent = spyPage

local filterLayout = Instance.new("UIListLayout")
filterLayout.FillDirection = Enum.FillDirection.Horizontal
filterLayout.Padding = UDim.new(0, 4)
filterLayout.SortOrder = Enum.SortOrder.LayoutOrder
filterLayout.Parent = filterBar

local filterButtons = {}
local CATEGORIES = {"ALL", "EMOTE", "WALK", "IDLE", "CUSTOM"}

for i, cat in ipairs(CATEGORIES) do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 66, 0, 26)
    btn.BackgroundColor3 = (cat == "ALL") and T.accent or T.card
    btn.TextColor3 = (cat == "ALL") and Color3.fromRGB(10,14,20) or T.subtext
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 9
    btn.Text = cat
    btn.AutoButtonColor = false
    btn.LayoutOrder = i
    btn.Parent = filterBar
    corner(btn, 5)
    stroke(btn, T.stroke, 1, 0.5)

    btn.MouseButton1Click:Connect(function()
        State.spyFilter = cat
        for c, b in pairs(filterButtons) do
            if c == cat then
                b.BackgroundColor3 = (c == "ALL") and T.accent or (CAT_COLORS[c] or T.accent)
                b.TextColor3 = Color3.fromRGB(10,14,20)
            else
                b.BackgroundColor3 = T.card
                b.TextColor3 = T.subtext
            end
        end
        -- Refresh l'affichage du spy
        for name, row in pairs(spyRows) do
            local data = State.spySeen[name]
            if data then
                row.Visible = (cat == "ALL") or (data.category == cat)
            end
        end
    end)

    filterButtons[cat] = btn
end

-- Liste SPY
local spyScroll = Instance.new("ScrollingFrame")
spyScroll.Size = UDim2.new(1, 0, 1, -146)
spyScroll.Position = UDim2.new(0, 0, 0, 142)
spyScroll.BackgroundColor3 = T.bg
spyScroll.BackgroundTransparency = 0.4
spyScroll.BorderSizePixel = 0
spyScroll.ScrollBarThickness = 3
spyScroll.ScrollBarImageColor3 = T.accent
spyScroll.ScrollBarImageTransparency = 0.5
spyScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
spyScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
spyScroll.Parent = spyPage
corner(spyScroll, 8)
stroke(spyScroll, T.stroke, 1, 0.5)

local spyLayout = Instance.new("UIListLayout")
spyLayout.Padding = UDim.new(0, 4)
spyLayout.SortOrder = Enum.SortOrder.LayoutOrder
spyLayout.Parent = spyScroll

local spyPad = Instance.new("UIPadding")
spyPad.PaddingTop = UDim.new(0, 6)
spyPad.PaddingBottom = UDim.new(0, 6)
spyPad.PaddingLeft = UDim.new(0, 6)
spyPad.PaddingRight = UDim.new(0, 6)
spyPad.Parent = spyScroll

local spyEmptyLbl = Instance.new("TextLabel")
spyEmptyLbl.Size = UDim2.new(1, -20, 0, 40)
spyEmptyLbl.Position = UDim2.new(0, 10, 0.5, -20)
spyEmptyLbl.BackgroundTransparency = 1
spyEmptyLbl.Text = "Aucune animation détectée...\nActive le SPY."
spyEmptyLbl.TextColor3 = T.dim
spyEmptyLbl.Font = Enum.Font.Gotham
spyEmptyLbl.TextSize = 10
spyEmptyLbl.TextWrapped = true
spyEmptyLbl.Parent = spyScroll

-- PAGE UTILS
local utilsPage = Instance.new("ScrollingFrame")
utilsPage.Size = UDim2.new(1, 0, 1, 0)
utilsPage.BackgroundTransparency = 1
utilsPage.BorderSizePixel = 0
utilsPage.ScrollBarThickness = 3
utilsPage.ScrollBarImageColor3 = T.accent
utilsPage.ScrollBarImageTransparency = 0.5
utilsPage.CanvasSize = UDim2.new(0, 0, 0, 0)
utilsPage.AutomaticCanvasSize = Enum.AutomaticSize.Y
utilsPage.Parent = content
utilsPage.Visible = false

local utilsLayout = Instance.new("UIListLayout")
utilsLayout.Padding = UDim.new(0, 6)
utilsLayout.SortOrder = Enum.SortOrder.LayoutOrder
utilsLayout.Parent = utilsPage

local utilsPad = Instance.new("UIPadding")
utilsPad.PaddingRight = UDim.new(0, 8)
utilsPad.Parent = utilsPage

local function makeSectionTitle(text)
    local holder = Instance.new("Frame")
    holder.Size = UDim2.new(1, 0, 0, 22)
    holder.BackgroundTransparency = 1
    holder.Parent = utilsPage
    local dot = Instance.new("Frame")
    dot.Size = UDim2.new(0, 6, 0, 6)
    dot.Position = UDim2.new(0, 2, 0.5, -3)
    dot.BackgroundColor3 = T.accent
    dot.BorderSizePixel = 0
    dot.Parent = holder
    corner(dot, 3)
    gradient(dot, T.accent, T.accent2, 45)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0, 150, 0, 22)
    lbl.Position = UDim2.new(0, 14, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3 = T.text
    lbl.Font = Enum.Font.GothamBold
    lbl.TextSize = 9
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Text = string.upper(text)
    lbl.Parent = holder
    local line = Instance.new("Frame")
    line.Size = UDim2.new(1, -180, 0, 1)
    line.Position = UDim2.new(0, 170, 0.5, 0)
    line.BackgroundColor3 = T.stroke
    line.BorderSizePixel = 0
    line.Parent = holder
    gradient3(line, T.stroke, T.bg, T.bg, 0)
end

local toggles = {}

local function makeUtilToggle(text, key, callback)
    local card = Instance.new("TextButton")
    card.Size = UDim2.new(1, 0, 0, 36)
    card.BackgroundColor3 = T.card
    card.BackgroundTransparency = 0.2
    card.Text = ""
    card.AutoButtonColor = false
    card.Parent = utilsPage
    corner(card, 8)
    local cardSt = stroke(card, T.stroke, 1, 0.5)

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -110, 1, 0)
    lbl.Position = UDim2.new(0, 14, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3 = T.text
    lbl.Font = Enum.Font.GothamMedium
    lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Text = text
    lbl.Parent = card

    local bindBtn = Instance.new("TextButton")
    bindBtn.Size = UDim2.new(0, 36, 0, 22)
    bindBtn.Position = UDim2.new(1, -92, 0.5, -11)
    bindBtn.BackgroundColor3 = T.bg
    bindBtn.TextColor3 = T.dim
    bindBtn.Font = Enum.Font.Code
    bindBtn.TextSize = 9
    bindBtn.Text = "---"
    bindBtn.AutoButtonColor = false
    bindBtn.Parent = card
    corner(bindBtn, 6)
    stroke(bindBtn, T.stroke, 1, 0.5)

    local track_ = Instance.new("Frame")
    track_.Size = UDim2.new(0, 38, 0, 20)
    track_.Position = UDim2.new(1, -50, 0.5, -10)
    track_.BackgroundColor3 = State[key] and T.accent or Color3.fromRGB(30,36,50)
    track_.BorderSizePixel = 0
    track_.Parent = card
    corner(track_, 10)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 16, 0, 16)
    knob.Position = State[key] and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
    knob.BackgroundColor3 = Color3.new(1,1,1)
    knob.BorderSizePixel = 0
    knob.Parent = track_
    corner(knob, 8)

    local function update()
        local on = State[key]
        tween(track_, 0.2, {BackgroundColor3 = on and T.accent or Color3.fromRGB(30,36,50)})
        tween(knob, 0.25, {Position = on and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8)}, Enum.EasingStyle.Back)
        tween(cardSt, 0.2, {Color = on and T.accent or T.stroke, Transparency = on and 0.3 or 0.5})
    end

    card.MouseButton1Click:Connect(function()
        State[key] = not State[key]
        update()
        if callback then callback(State[key]) end
    end)

    card.MouseEnter:Connect(function() tween(card, 0.15, {BackgroundColor3 = T.cardHover, BackgroundTransparency = 0}) end)
    card.MouseLeave:Connect(function() tween(card, 0.15, {BackgroundColor3 = T.card, BackgroundTransparency = 0.2}) end)

    bindBtn.MouseButton1Click:Connect(function()
        if BindWaiting == key then
            BindWaiting = nil
            bindBtn.Text = Binds[key] and Binds[key].Name or "---"
            bindBtn.TextColor3 = Binds[key] and T.accent or T.dim
            return
        end
        BindWaiting = key
        bindBtn.Text = "?"
        bindBtn.TextColor3 = T.yellow
    end)

    bindBtn.MouseButton2Click:Connect(function()
        Binds[key] = nil
        bindBtn.Text = "---"
        bindBtn.TextColor3 = T.dim
    end)

    toggles[key] = {update = update, callback = callback, bindBtn = bindBtn}
    return card
end

-- CHAMS
local function clearChams()
    for _, hl in pairs(chamsInstances) do pcall(function() hl:Destroy() end) end
    chamsInstances = {}
end

local function applyChamsForPlayer(plr)
    if plr == lp then return end
    local char = plr.Character
    if not char then return end
    local role = getPlayerRole(plr)
    local color = ROLE_COLORS[role] or T.accent
    if chamsInstances[plr.Name] then
        local hl = chamsInstances[plr.Name]
        if hl and hl.Parent then
            hl.FillColor = color
            hl.OutlineColor = color
            return
        end
    end
    local hl = Instance.new("Highlight")
    hl.Name = "MM2_Chams"
    hl.Adornee = char
    hl.FillColor = color
    hl.OutlineColor = color
    hl.FillTransparency = 0.5
    hl.OutlineTransparency = 0
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.Parent = char
    chamsInstances[plr.Name] = hl
end

local function refreshChams()
    if not State.chams then clearChams(); return end
    for name, hl in pairs(chamsInstances) do
        local plr = Players:FindFirstChild(name)
        if not plr or not plr.Character or hl.Parent ~= plr.Character then
            pcall(function() hl:Destroy() end)
            chamsInstances[name] = nil
        end
    end
    for _, plr in ipairs(Players:GetPlayers()) do
        applyChamsForPlayer(plr)
    end
end

-- GUN ESP
local function clearGunESP()
    for _, hl in pairs(gunHighlights) do pcall(function() hl:Destroy() end) end
    gunHighlights = {}
end

local function applyGunESPForPart(part)
    if not part or not part.Parent or gunHighlights[part] then return end
    local hl = Instance.new("Highlight")
    hl.Name = "MM2_GunESP"
    hl.Adornee = part
    hl.FillColor = T.gold
    hl.OutlineColor = T.gold
    hl.FillTransparency = 0.4
    hl.OutlineTransparency = 0
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.Parent = part
    gunHighlights[part] = hl
end

local function refreshGunESP()
    clearGunESP()
    if not State.gunESP then return end
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") and obj.Name == "GunDrop" then
            applyGunESPForPart(obj)
        end
    end
end

-- AUTO GRAB
local function findDroppedGun()
    for _, obj in ipairs(workspace:GetChildren()) do
        if obj:IsA("BasePart") and obj.Name == "GunDrop" then return obj end
        if obj:IsA("Model") then
            local g = obj:FindFirstChild("GunDrop")
            if g and g:IsA("BasePart") then return g end
        end
    end
    return nil
end

local function grabGun(gun)
    if State.grabbing or not gun or not gun.Parent then return end
    local char = lp.Character
    if not char then return end
    if char:FindFirstChild("Gun") then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    State.grabbing = true
    pcall(function()
        gun.CFrame = hrp.CFrame
        gun.AssemblyLinearVelocity = Vector3.zero
        gun.AssemblyAngularVelocity = Vector3.zero
    end)
    if type(firetouchinterest) == "function" then
        pcall(function()
            firetouchinterest(hrp, gun, 0)
            task.wait(0.05)
            firetouchinterest(hrp, gun, 1)
        end)
    end
    task.wait(0.1)
    State.grabbing = false
end

local function autoGrabTick()
    if not State.autoGrab then return end
    if State.grabbing then return end
    local char = lp.Character
    if not char then return end
    if char:FindFirstChild("Gun") then return end
    local now = tick()
    if now - State.lastGrabAttempt < 0.5 then return end
    local gun = findDroppedGun()
    if gun then
        State.lastGrabAttempt = now
        task.spawn(grabGun, gun)
    end
end

track(workspace.DescendantAdded:Connect(function(d)
    if State.gunESP and d:IsA("BasePart") and d.Name == "GunDrop" then
        task.defer(applyGunESPForPart, d)
    end
    if State.autoGrab and d:IsA("BasePart") and d.Name == "GunDrop" then
        task.spawn(grabGun, d)
    end
end))

local function applySpeed()
    local hum = lp.Character and lp.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    hum.WalkSpeed = State.speed and State.speedValue or State.defaultSpeed
end
local statusLbl
-- ============ SERVER HOP ============
local HOP_FILE = "NotSameServers.json"
local hopVisited = {}
local hopCursor = ""
local hopHour = os.date("!*t").hour
local hopInProgress = false

-- Init du fichier de mémoire
local function hopInitFile()
    if not isfile or not readfile or not writefile then
        hopVisited = { hopHour }
        return
    end
    pcall(function()
        if isfile(HOP_FILE) then
            local ok, decoded = pcall(function()
                return HttpService:JSONDecode(readfile(HOP_FILE))
            end)
            if ok and type(decoded) == "table" then
                hopVisited = decoded
            else
                hopVisited = { hopHour }
                writefile(HOP_FILE, HttpService:JSONEncode(hopVisited))
            end
        else
            hopVisited = { hopHour }
            writefile(HOP_FILE, HttpService:JSONEncode(hopVisited))
        end
    end)
end

local function hopSave()
    if not writefile then return end
    pcall(function()
        writefile(HOP_FILE, HttpService:JSONEncode(hopVisited))
    end)
end

-- Un hop, retourne (true, infos) ou (false, raison)
local function hopToServer()
    if hopInProgress then return false, "Hop en cours" end
    hopInProgress = true

    hopInitFile()
    hopCursor = ""

    local TeleportService = game:GetService("TeleportService")
    local attempts = 0
    local maxAttempts = 3

    while attempts < maxAttempts do
        attempts = attempts + 1

        -- URL avec ou sans curseur
        local url = "https://games.roblox.com/v1/games/" .. game.PlaceId ..
                    "/servers/Public?sortOrder=Asc&limit=100"
        if hopCursor ~= "" then
            url = url .. "&cursor=" .. hopCursor
        end

        -- Récupère la liste
        local ok, body = pcall(function() return game:HttpGet(url) end)
        if not ok or not body then
            hopInProgress = false
            return false, "HTTP bloqué (executor ?)"
        end

        local ok2, site = pcall(function() return HttpService:JSONDecode(body) end)
        if not ok2 or not site or not site.data then
            hopInProgress = false
            return false, "Réponse API invalide"
        end

        -- Met à jour le curseur pour les prochaines pages
        if site.nextPageCursor and site.nextPageCursor ~= "null" and site.nextPageCursor ~= nil then
            hopCursor = site.nextPageCursor
        else
            hopCursor = ""
        end

        -- Cherche un serveur dispo
        for _, srv in ipairs(site.data) do
            local id = tostring(srv.id)
            if tonumber(srv.maxPlayers) > tonumber(srv.playing) and id ~= game.JobId then
                local alreadyVisited = false
                for _, existing in ipairs(hopVisited) do
                    if id == tostring(existing) then
                        alreadyVisited = true
                        break
                    end
                end

                if not alreadyVisited then
                    table.insert(hopVisited, id)
                    hopSave()

                    local okTp = pcall(function()
                        TeleportService:TeleportToPlaceInstance(game.PlaceId, id, lp)
                    end)

                    if okTp then
                        hopInProgress = false
                        return true, srv.playing .. "/" .. srv.maxPlayers
                    end
                end
            end
        end

        -- Pas trouvé sur cette page → essaie la suivante
        if hopCursor == "" then break end
    end

    hopInProgress = false
    return false, "Aucun serveur dispo"
end

-- TOGGLES
makeSectionTitle("Utilitaires")
makeUtilToggle("CHAMS (rôle)", "chams", function() refreshChams() end)
makeUtilToggle("GUN ESP", "gunESP", function() refreshGunESP() end)
makeUtilToggle("AUTO GRAB GUN", "autoGrab", function(on) if on then task.spawn(autoGrabTick) end end)
makeUtilToggle("SPEED 19", "speed", function() applySpeed() end)

makeSectionTitle("Aim Lock")
makeUtilToggle("AIM LOCK (Murderer)", "aimLock", function() State.aimCurrentTarget = nil end)

makeSectionTitle("Server")
local hopBtn = Instance.new("TextButton")
hopBtn.Size = UDim2.new(1, 0, 0, 36)
hopBtn.BackgroundColor3 = Color3.fromRGB(20, 40, 60)
hopBtn.TextColor3 = T.accent
hopBtn.Font = Enum.Font.GothamBold
hopBtn.TextSize = 11
hopBtn.Text = "🔄 SERVER HOP"
hopBtn.AutoButtonColor = false
hopBtn.Parent = utilsPage
corner(hopBtn, 8)
stroke(hopBtn, T.accent, 1, 0.4)

hopBtn.MouseButton1Click:Connect(function()
    if hopInProgress then return end
    hopBtn.Text = "🔄 HOP..."
    statusLbl.Text = "Recherche serveur..."
    statusLbl.TextColor3 = T.accent

    task.spawn(function()
        local ok, info = hopToServer()
        if ok then
            statusLbl.Text = "🔄 Hop -> " .. info
            statusLbl.TextColor3 = T.green
        else
            statusLbl.Text = "❌ " .. tostring(info)
            statusLbl.TextColor3 = T.red
            hopBtn.Text = "🔄 SERVER HOP"
        end
    end)
end)

local rejoinBtn = Instance.new("TextButton")
rejoinBtn.Size = UDim2.new(1, 0, 0, 32)
rejoinBtn.BackgroundColor3 = T.card
rejoinBtn.TextColor3 = T.yellow
rejoinBtn.Font = Enum.Font.GothamBold
rejoinBtn.TextSize = 11
rejoinBtn.Text = "🔁 REJOIN (même serveur)"
rejoinBtn.AutoButtonColor = false
rejoinBtn.Parent = utilsPage
corner(rejoinBtn, 8)
stroke(rejoinBtn, T.yellow, 1, 0.4)

rejoinBtn.MouseButton1Click:Connect(function()
    local TeleportService = game:GetService("TeleportService")
    statusLbl.Text = "🔁 Rejoin..."
    statusLbl.TextColor3 = T.yellow
    task.wait(0.3)
    pcall(function()
        TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, lp)
    end)
end)

local modeFrame = Instance.new("Frame")
modeFrame.Size = UDim2.new(1, 0, 0, 36)
modeFrame.BackgroundColor3 = T.card
modeFrame.BackgroundTransparency = 0.2
modeFrame.BorderSizePixel = 0
modeFrame.Parent = utilsPage
corner(modeFrame, 8)
stroke(modeFrame, T.stroke, 1, 0.5)

local modeLbl = Instance.new("TextLabel")
modeLbl.Size = UDim2.new(0, 60, 1, 0)
modeLbl.Position = UDim2.new(0, 14, 0, 0)
modeLbl.BackgroundTransparency = 1
modeLbl.Text = "MODE"
modeLbl.TextColor3 = T.subtext
modeLbl.Font = Enum.Font.GothamBold
modeLbl.TextSize = 9
modeLbl.TextXAlignment = Enum.TextXAlignment.Left
modeLbl.Parent = modeFrame

local modeBtn = Instance.new("TextButton")
modeBtn.Size = UDim2.new(0, 130, 0, 26)
modeBtn.Position = UDim2.new(1, -144, 0.5, -13)
modeBtn.BackgroundColor3 = T.accent
modeBtn.TextColor3 = Color3.fromRGB(10,14,20)
modeBtn.Font = Enum.Font.GothamBold
modeBtn.TextSize = 9
modeBtn.Text = "SMOOTH"
modeBtn.AutoButtonColor = false
modeBtn.Parent = modeFrame
corner(modeBtn, 6)
gradient(modeBtn, T.accent, T.accent2, 0)

modeBtn.MouseButton1Click:Connect(function()
    if State.aimMode == "SMOOTH" then
        State.aimMode = "SNAP"
        modeBtn.Text = "SNAP"
        modeBtn.BackgroundColor3 = T.red
    elseif State.aimMode == "SNAP" then
        State.aimMode = "PREDICT"
        modeBtn.Text = "PREDICT"
        modeBtn.BackgroundColor3 = T.green
    else
        State.aimMode = "SMOOTH"
        modeBtn.Text = "SMOOTH"
        modeBtn.BackgroundColor3 = T.accent
    end
    State.aimCurrentTarget = nil
end)

-- BOTTOM
local bottomBar = Instance.new("Frame")
bottomBar.Size = UDim2.new(1, -28, 0, 36)
bottomBar.Position = UDim2.new(0, 14, 1, -80)
bottomBar.BackgroundColor3 = T.card
bottomBar.BackgroundTransparency = 0.3
bottomBar.BorderSizePixel = 0
bottomBar.Parent = panel
corner(bottomBar, 8)
stroke(bottomBar, T.stroke, 1, 0.5)

local stopBtn = Instance.new("TextButton")
stopBtn.Size = UDim2.new(0.5, -5, 1, -6)
stopBtn.Position = UDim2.new(0, 3, 0, 3)
stopBtn.BackgroundColor3 = Color3.fromRGB(60, 22, 35)
stopBtn.TextColor3 = T.red
stopBtn.Font = Enum.Font.GothamBold
stopBtn.TextSize = 10
stopBtn.Text = "STOP EMOTE"
stopBtn.AutoButtonColor = false
stopBtn.Parent = bottomBar
corner(stopBtn, 6)

statusLbl = Instance.new("TextLabel")
statusLbl.Size = UDim2.new(0.5, -5, 1, -6)
statusLbl.Position = UDim2.new(0.5, 2, 0, 3)
statusLbl.BackgroundColor3 = T.bg
statusLbl.BackgroundTransparency = 0.3
statusLbl.TextColor3 = T.subtext
statusLbl.Font = Enum.Font.Code
statusLbl.TextSize = 9
statusLbl.Text = "aucune"
statusLbl.Parent = bottomBar
corner(statusLbl, 6)

local eraseBtn = Instance.new("TextButton")
eraseBtn.Size = UDim2.new(1, -28, 0, 34)
eraseBtn.Position = UDim2.new(0, 14, 1, -40)
eraseBtn.BackgroundColor3 = Color3.fromRGB(100, 22, 38)
eraseBtn.BackgroundTransparency = 0.1
eraseBtn.TextColor3 = Color3.fromRGB(255, 220, 225)
eraseBtn.Font = Enum.Font.GothamBold
eraseBtn.TextSize = 11
eraseBtn.Text = "ERASE - UNLOAD"
eraseBtn.AutoButtonColor = false
eraseBtn.Parent = panel
corner(eraseBtn, 8)
local eraseSt = stroke(eraseBtn, T.red, 1, 0.35)
gradient(eraseBtn, Color3.fromRGB(180, 40, 60), Color3.fromRGB(100, 22, 38), 0)

-- LOGIQUE EMOTE
local function getAnimator()
    local char = lp.Character
    if not char then return nil end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return nil end
    local animator = hum:FindFirstChildOfClass("Animator")
    if not animator then
        animator = Instance.new("Animator")
        animator.Parent = hum
    end
    return animator
end

local function playEmote(id, name)
    local animator = getAnimator()
    if not animator then return false end
    id = normalizeAnimId(id)
    if not id then return false end
    id = "rbxassetid://" .. id
    if State.emoteTrack then
        pcall(function() State.emoteTrack:Stop(0.1) end)
        State.emoteTrack = nil
    end
    local anim = Instance.new("Animation")
    anim.AnimationId = id
    local ok, newTrack = pcall(function() return animator:LoadAnimation(anim) end)
    if not ok or not newTrack then return false end
    newTrack.Priority = Enum.AnimationPriority.Action4
    newTrack.Looped = true
    newTrack:Play(0.1)
    State.emoteTrack = newTrack
    State.currentId = id
    return true
end

local function stopEmote(clearCurrent)
    if State.emoteTrack then
        pcall(function() State.emoteTrack:Stop(0.2) end)
        State.emoteTrack = nil
    end
    State.pendingId = nil
    if clearCurrent then
        State.currentId = nil
        statusLbl.Text = "aucune"
        statusLbl.TextColor3 = T.subtext
    end
end

local function isMoving()
    local K = Enum.KeyCode
    return UserInputService:IsKeyDown(K.W) or UserInputService:IsKeyDown(K.A)
        or UserInputService:IsKeyDown(K.S) or UserInputService:IsKeyDown(K.D)
        or UserInputService:IsKeyDown(K.Up) or UserInputService:IsKeyDown(K.Down)
        or UserInputService:IsKeyDown(K.Left) or UserInputService:IsKeyDown(K.Right)
end

track(RunService.Heartbeat:Connect(function()
    local hum = lp.Character and lp.Character:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then
        if State.emoteTrack then
            pcall(function() State.emoteTrack:Stop(0.2) end)
            State.emoteTrack = nil
        end
        return
    end
    if isMoving() then
        if State.emoteTrack and State.emoteTrack.IsPlaying then
            pcall(function() State.emoteTrack:Stop(0.15) end)
        end
        return
    end
    if State.autoEmote then
        if State.pendingId then
            local id = State.pendingId
            State.pendingId = nil
            playEmote(id)
            return
        end
        if State.currentId and (not State.emoteTrack or not State.emoteTrack.IsPlaying) then
            playEmote(State.currentId)
        end
    end
end))

track(RunService.Heartbeat:Connect(function()
    if tick() - lastRoleRefresh > 1 then
        lastRoleRefresh = tick()
        task.spawn(updateRoleCache)
    end
    if State.chams and tick() - lastChamsRefresh > 0.5 then
        lastChamsRefresh = tick()
        refreshChams()
    end
    if State.autoGrab then
        autoGrabTick()
    end
end))

-- LIBRARY
local function refreshLibrary()
    for _, c in ipairs(libraryPage:GetChildren()) do
        if c:IsA("TextButton") then c:Destroy() end
    end
    libraryRows = {}
    local idx = 0
    for name, data in pairs(LIBRARY) do
        idx = idx + 1
        local id = data.id
        local color = T.accent
        if type(data.color) == "table" and #data.color >= 3 then
            color = Color3.fromRGB(data.color[1]*255, data.color[2]*255, data.color[3]*255)
        end

        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 106, 0, 62)
        btn.BackgroundColor3 = T.card
        btn.BackgroundTransparency = 0.15
        btn.Text = ""
        btn.AutoButtonColor = false
        btn.LayoutOrder = idx
        btn.Parent = libraryPage
        corner(btn, 10)
        local st = stroke(btn, T.stroke, 1, 0.5)

        local accentBar = Instance.new("Frame")
        accentBar.Size = UDim2.new(0, 3, 0.55, 0)
        accentBar.Position = UDim2.new(0, 6, 0.5, 0)
        accentBar.BackgroundColor3 = color
        accentBar.BorderSizePixel = 0
        accentBar.BackgroundTransparency = 0.15
        accentBar.Parent = btn
        corner(accentBar, 2)

        local lbl = Instance.new("TextLabel")
        lbl.Size = UDim2.new(1, -18, 0, 16)
        lbl.Position = UDim2.new(0, 16, 0, 12)
        lbl.BackgroundTransparency = 1
        lbl.Text = name
        lbl.TextColor3 = T.text
        lbl.Font = Enum.Font.GothamBold
        lbl.TextSize = 11
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.TextTruncate = Enum.TextTruncate.AtEnd
        lbl.Parent = btn

        local idLbl = Instance.new("TextLabel")
        idLbl.Size = UDim2.new(1, -18, 0, 10)
        idLbl.Position = UDim2.new(0, 16, 0, 32)
        idLbl.BackgroundTransparency = 1
        idLbl.Text = tostring(id):gsub("rbxassetid://", "")
        idLbl.TextColor3 = T.dim
        idLbl.Font = Enum.Font.Code
        idLbl.TextSize = 8
        idLbl.TextXAlignment = Enum.TextXAlignment.Left
        idLbl.TextTruncate = Enum.TextTruncate.AtEnd
        idLbl.Parent = btn

        btn.MouseEnter:Connect(function()
            tween(btn, 0.18, {BackgroundColor3 = T.cardHover, BackgroundTransparency = 0})
            tween(st, 0.18, {Color = color, Transparency = 0.15})
        end)
        btn.MouseLeave:Connect(function()
            tween(btn, 0.18, {BackgroundColor3 = T.card, BackgroundTransparency = 0.15})
            tween(st, 0.18, {Color = T.stroke, Transparency = 0.5})
        end)
        btn.MouseButton1Click:Connect(function()
            State.currentId = id
            State.pendingId = id
            statusLbl.Text = name
            statusLbl.TextColor3 = color
            if not isMoving() then
                playEmote(id, name)
                State.pendingId = nil
            end
        end)
        btn.MouseButton2Click:Connect(function()
            LIBRARY[name] = nil
            saveLibrary()
            refreshLibrary()
        end)

        libraryRows[name] = btn
    end
    libraryEmptyLbl.Visible = (idx == 0)
end

-- SPY
local spyRows = {}

local function clearSpyRow(name)
    if spyRows[name] then
        pcall(function() spyRows[name]:Destroy() end)
        spyRows[name] = nil
    end
end

local function updateSpyEmpty()
    local hasRows = false
    for _ in pairs(spyRows) do hasRows = true; break end
    spyEmptyLbl.Visible = not hasRows
end

-- Prompt
local savePrompt = Instance.new("Frame")
savePrompt.Size = UDim2.new(0, 320, 0, 140)
savePrompt.Position = UDim2.new(0.5, -160, 0.5, -70)
savePrompt.BackgroundColor3 = T.card
savePrompt.BorderSizePixel = 0
savePrompt.Parent = gui
savePrompt.Visible = false
corner(savePrompt, 10)
stroke(savePrompt, T.accent, 1.2, 0.3)

local savePromptTitle = Instance.new("TextLabel")
savePromptTitle.Size = UDim2.new(1, -20, 0, 24)
savePromptTitle.Position = UDim2.new(0, 10, 0, 8)
savePromptTitle.BackgroundTransparency = 1
savePromptTitle.Text = "Nom de l'emote :"
savePromptTitle.TextColor3 = T.text
savePromptTitle.Font = Enum.Font.GothamBold
savePromptTitle.TextSize = 12
savePromptTitle.TextXAlignment = Enum.TextXAlignment.Left
savePromptTitle.Parent = savePrompt

local savePromptId = Instance.new("TextLabel")
savePromptId.Size = UDim2.new(1, -20, 0, 14)
savePromptId.Position = UDim2.new(0, 10, 0, 30)
savePromptId.BackgroundTransparency = 1
savePromptId.Text = "ID: "
savePromptId.TextColor3 = T.dim
savePromptId.Font = Enum.Font.Code
savePromptId.TextSize = 9
savePromptId.TextXAlignment = Enum.TextXAlignment.Left
savePromptId.Parent = savePrompt

local savePromptInput = Instance.new("TextBox")
savePromptInput.Size = UDim2.new(1, -20, 0, 28)
savePromptInput.Position = UDim2.new(0, 10, 0, 48)
savePromptInput.BackgroundColor3 = T.bg
savePromptInput.Text = ""
savePromptInput.PlaceholderText = "Ex: MonDab"
savePromptInput.PlaceholderColor3 = T.dim
savePromptInput.TextColor3 = T.text
savePromptInput.Font = Enum.Font.Code
savePromptInput.TextSize = 12
savePromptInput.ClearTextOnFocus = false
savePromptInput.Parent = savePrompt
corner(savePromptInput, 6)
stroke(savePromptInput, T.stroke, 1, 0.5)

local savePromptOk = Instance.new("TextButton")
savePromptOk.Size = UDim2.new(0.5, -15, 0, 28)
savePromptOk.Position = UDim2.new(0, 10, 1, -36)
savePromptOk.BackgroundColor3 = Color3.fromRGB(20, 60, 40)
savePromptOk.TextColor3 = T.green
savePromptOk.Font = Enum.Font.GothamBold
savePromptOk.TextSize = 11
savePromptOk.Text = "SAUVEGARDER"
savePromptOk.AutoButtonColor = false
savePromptOk.Parent = savePrompt
corner(savePromptOk, 6)

local savePromptCancel = Instance.new("TextButton")
savePromptCancel.Size = UDim2.new(0.5, -15, 0, 28)
savePromptCancel.Position = UDim2.new(0.5, 5, 1, -36)
savePromptCancel.BackgroundColor3 = Color3.fromRGB(60, 20, 30)
savePromptCancel.TextColor3 = T.red
savePromptCancel.Font = Enum.Font.GothamBold
savePromptCancel.TextSize = 11
savePromptCancel.Text = "ANNULER"
savePromptCancel.AutoButtonColor = false
savePromptCancel.Parent = savePrompt
corner(savePromptCancel, 6)

local pendingSaveId = nil

local function openSavePrompt(id)
    pendingSaveId = id
    savePromptInput.Text = ""
    savePromptId.Text = "ID: " .. tostring(id)
    savePrompt.Visible = true
end

savePromptCancel.MouseButton1Click:Connect(function()
    savePrompt.Visible = false
    pendingSaveId = nil
end)

savePromptOk.MouseButton1Click:Connect(function()
    local name = savePromptInput.Text
    name = name:gsub("[^%w%s_%-]", ""):gsub("^%s+", ""):gsub("%s+$", "")
    if name == "" or not pendingSaveId then return end
    LIBRARY[name] = {
        id = pendingSaveId,
        color = {T.accent.R, T.accent.G, T.accent.B}
    }
    saveLibrary()
    refreshLibrary()
    savePrompt.Visible = false
    statusLbl.Text = "✅ Sauvegardé : " .. name
    statusLbl.TextColor3 = T.green
    pendingSaveId = nil
end)

-- createSpyRow avec badge catégorie
local function createSpyRow(key, playerName, emoteName, emoteId, remaining, category)
    local catColor = CAT_COLORS[category] or T.accent
    if spyRows[key] then
        local row = spyRows[key]
        local emoteLbl = row:FindFirstChild("EmoteLbl")
        local idLbl = row:FindFirstChild("IdLbl")
        local timerLbl = row:FindFirstChild("TimerLbl")
        local catBadge = row:FindFirstChild("CatBadge")
        if emoteLbl then emoteLbl.Text = emoteName end
        if idLbl then idLbl.Text = emoteId end
        if timerLbl then timerLbl.Text = tostring(remaining or 10) .. "s" end
        if catBadge then
            catBadge.Text = category or "?"
            catBadge.TextColor3 = catColor
            catBadge.BackgroundColor3 = catColor
        end
        -- Applique le filtre
        row.Visible = (State.spyFilter == "ALL") or (State.spyFilter == category)
        return
    end

    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 34)
    row.BackgroundColor3 = T.card
    row.BackgroundTransparency = 0.2
    row.BorderSizePixel = 0
    row.Parent = spyScroll
    corner(row, 6)
    stroke(row, T.stroke, 1, 0.5)
    row.Visible = (State.spyFilter == "ALL") or (State.spyFilter == category)

    -- Badge catégorie
    local catBadge = Instance.new("TextLabel")
    catBadge.Name = "CatBadge"
    catBadge.Size = UDim2.new(0, 50, 0, 16)
    catBadge.Position = UDim2.new(0, 8, 0.5, -8)
    catBadge.BackgroundColor3 = catColor
    catBadge.BackgroundTransparency = 0.75
    catBadge.TextColor3 = catColor
    catBadge.Font = Enum.Font.GothamBold
    catBadge.TextSize = 8
    catBadge.Text = category or "?"
    catBadge.Parent = row
    corner(catBadge, 4)

    local nameLbl = Instance.new("TextLabel")
    nameLbl.Name = "NameLbl"
    nameLbl.Size = UDim2.new(0, 68, 1, 0)
    nameLbl.Position = UDim2.new(0, 62, 0, 0)
    nameLbl.BackgroundTransparency = 1
    nameLbl.Text = playerName
    nameLbl.TextColor3 = T.text
    nameLbl.Font = Enum.Font.GothamBold
    nameLbl.TextSize = 9
    nameLbl.TextXAlignment = Enum.TextXAlignment.Left
    nameLbl.TextTruncate = Enum.TextTruncate.AtEnd
    nameLbl.Parent = row

    local emoteLbl = Instance.new("TextLabel")
    emoteLbl.Name = "EmoteLbl"
    emoteLbl.Size = UDim2.new(0, 60, 1, 0)
    emoteLbl.Position = UDim2.new(0, 134, 0, 0)
    emoteLbl.BackgroundTransparency = 1
    emoteLbl.Text = emoteName
    emoteLbl.TextColor3 = T.accent
    emoteLbl.Font = Enum.Font.GothamMedium
    emoteLbl.TextSize = 9
    emoteLbl.TextXAlignment = Enum.TextXAlignment.Left
    emoteLbl.TextTruncate = Enum.TextTruncate.AtEnd
    emoteLbl.Parent = row

    local timerLbl = Instance.new("TextLabel")
    timerLbl.Name = "TimerLbl"
    timerLbl.Size = UDim2.new(0, 22, 1, 0)
    timerLbl.Position = UDim2.new(0, 198, 0, 0)
    timerLbl.BackgroundTransparency = 1
    timerLbl.Text = tostring(remaining or 10) .. "s"
    timerLbl.TextColor3 = T.dim
    timerLbl.Font = Enum.Font.Code
    timerLbl.TextSize = 9
    timerLbl.TextXAlignment = Enum.TextXAlignment.Center
    timerLbl.Parent = row

    local idLbl = Instance.new("TextLabel")
    idLbl.Name = "IdLbl"
    idLbl.Size = UDim2.new(0, 54, 1, 0)
    idLbl.Position = UDim2.new(0, 222, 0, 0)
    idLbl.BackgroundTransparency = 1
    idLbl.Text = emoteId
    idLbl.TextColor3 = T.dim
    idLbl.Font = Enum.Font.Code
    idLbl.TextSize = 8
    idLbl.TextXAlignment = Enum.TextXAlignment.Left
    idLbl.TextTruncate = Enum.TextTruncate.AtEnd
    idLbl.Parent = row

    -- SAVE
    local saveBtn = Instance.new("TextButton")
    saveBtn.Size = UDim2.new(0, 22, 0, 22)
    saveBtn.Position = UDim2.new(1, -130, 0.5, -11)
    saveBtn.BackgroundColor3 = T.green
    saveBtn.TextColor3 = Color3.fromRGB(10,14,20)
    saveBtn.Font = Enum.Font.GothamBold
    saveBtn.TextSize = 10
    saveBtn.Text = "+"
    saveBtn.AutoButtonColor = false
    saveBtn.Parent = row
    corner(saveBtn, 5)

    -- COPY
    local copyBtn = Instance.new("TextButton")
    copyBtn.Size = UDim2.new(0, 36, 0, 22)
    copyBtn.Position = UDim2.new(1, -106, 0.5, -11)
    copyBtn.BackgroundColor3 = T.bg
    copyBtn.TextColor3 = T.accent
    copyBtn.Font = Enum.Font.Code
    copyBtn.TextSize = 8
    copyBtn.Text = "COPY"
    copyBtn.AutoButtonColor = false
    copyBtn.Parent = row
    corner(copyBtn, 5)
    stroke(copyBtn, T.stroke, 1, 0.5)

    -- PLAY
    local playBtn = Instance.new("TextButton")
    playBtn.Size = UDim2.new(0, 36, 0, 22)
    playBtn.Position = UDim2.new(1, -68, 0.5, -11)
    playBtn.BackgroundColor3 = T.accent
    playBtn.TextColor3 = Color3.fromRGB(10,14,20)
    playBtn.Font = Enum.Font.GothamBold
    playBtn.TextSize = 8
    playBtn.Text = "▶"
    playBtn.AutoButtonColor = false
    playBtn.Parent = row
    corner(playBtn, 5)
    gradient(playBtn, T.accent, T.accent2, 0)

    saveBtn.MouseButton1Click:Connect(function()
        openSavePrompt(emoteId)
    end)

    copyBtn.MouseButton1Click:Connect(function()
        if setclipboard then
            setclipboard(emoteId)
            copyBtn.Text = "✓"
            task.wait(1)
            copyBtn.Text = "COPY"
        end
    end)

    playBtn.MouseButton1Click:Connect(function()
        playEmote(emoteId, emoteName)
        statusLbl.Text = emoteName
        statusLbl.TextColor3 = T.accent
    end)

    spyRows[key] = row
    updateSpyEmpty()
end

-- spyTick CORRIGÉ
local function spyTick()
    if not State.spyEnabled then return end

    local now = tick()
    local activePlayers = {}

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= lp and plr.Character then
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if hum and hum.Health > 0 then
                local animator = hum:FindFirstChildOfClass("Animator")
                if animator then
                    local foundEmote = nil
                    pcall(function()
                        local tracks = animator:GetPlayingAnimationTracks()
                        for _, t in ipairs(tracks) do
                            if t.IsPlaying and t.Animation then
                                local prio = t.Priority
                                local len = t.Length or 0
                                -- Ignore Idle
                                local isIdle = (prio == Enum.AnimationPriority.Idle)
                                -- Ignore Core court (locomotion)
                                local isCoreShort = (prio == Enum.AnimationPriority.Core and len < 1.0)
                                if not isIdle and not isCoreShort then
                                    if len >= 0.5 then
                                        local id = t.Animation.AnimationId
                                        local animName = t.Animation.Name or ""
                                        if id and id ~= "" and not isExcludedName(animName) then
                                            local cleanId = normalizeAnimId(id)
                                            local name = EMOTE_DB["rbxassetid://" .. cleanId] or EMOTE_DB[cleanId]
                                            local cat = categorizeAnim(prio, len)
                                            foundEmote = {
                                                id = "rbxassetid://" .. cleanId,
                                                name = name,
                                                animName = animName,
                                                length = len,
                                                category = cat
                                            }
                                            break
                                        end
                                    end
                                end
                            end
                        end
                    end)
                      if foundEmote then
                        activePlayers[plr.Name] = true
                        -- Clé unique : playerName + id
                        local key = plr.Name .. "|" .. foundEmote.id
                        local entry = State.spySeen[key]
                        if not entry then
                            State.spySeen[key] = {
                                playerName = plr.Name,
                                id = foundEmote.id,
                                name = foundEmote.name,
                                animName = foundEmote.animName,
                                length = foundEmote.length,
                                category = foundEmote.category,
                                lastSeen = now,
                            }
                        else
                            entry.lastSeen = now
                        end
                    end
                end
            end
        end
    end

        for key, data in pairs(State.spySeen) do
        local elapsed = now - data.lastSeen
        if elapsed > State.spyTimeout then
            clearSpyRow(key)
            State.spySeen[key] = nil
        else
            local displayName = data.name
            if not displayName then
                if data.animName and data.animName ~= "" and data.animName ~= "Animation" then
                    displayName = data.animName
                else
                    local shortId = data.id:gsub("rbxassetid://", "")
                    if #shortId > 8 then shortId = shortId:sub(1, 8) .. "..." end
                    displayName = "ID:" .. shortId
                end
            end
            local remaining = math.ceil(State.spyTimeout - elapsed)
            createSpyRow(key, data.playerName, displayName, data.id, remaining, data.category or "CUSTOM")
        end
    end

    updateSpyEmpty()
end

track(RunService.Heartbeat:Connect(function()
    if not State.spyEnabled then return end
    if tick() - lastSpyRefresh < 0.5 then return end
    lastSpyRefresh = tick()
    spyTick()
end))

-- SPY TOGGLE
spyToggleBtn.MouseButton1Click:Connect(function()
    State.spyEnabled = not State.spyEnabled
    if State.spyEnabled then
        spyToggleBtn.Text = "SPY : ON"
        spyToggleBtn.TextColor3 = T.green
        spyToggleBtn.BackgroundColor3 = Color3.fromRGB(20, 50, 40)
        task.spawn(spyTick)
    else
        spyToggleBtn.Text = "SPY : OFF"
        spyToggleBtn.TextColor3 = T.subtext
        spyToggleBtn.BackgroundColor3 = T.card
        for name, _ in pairs(spyRows) do clearSpyRow(name) end
        updateSpyEmpty()
    end
end)

-- CUSTOM PLAY
customPlayBtn.MouseButton1Click:Connect(function()
    local id = customInput.Text
    if id == "" then return end
    playEmote(id, "Custom")
    statusLbl.Text = "Custom"
    statusLbl.TextColor3 = T.accent
end)

-- EMOTE BUTTONS
for i, e in ipairs(EMOTES) do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 106, 0, 62)
    btn.BackgroundColor3 = T.card
    btn.BackgroundTransparency = 0.15
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.LayoutOrder = i
    btn.Parent = emotesPage
    corner(btn, 10)
    local btnSt = stroke(btn, T.stroke, 1, 0.5)
    local accentBar = Instance.new("Frame")
    accentBar.Size = UDim2.new(0, 3, 0.55, 0)
    accentBar.Position = UDim2.new(0, 6, 0.5, 0)
    accentBar.BackgroundColor3 = e.color
    accentBar.BorderSizePixel = 0
    accentBar.BackgroundTransparency = 0.15
    accentBar.Parent = btn
    corner(accentBar, 2)
    gradient(accentBar, e.color, e.color:Lerp(Color3.new(0,0,0), 0.5), 90)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -18, 0, 16)
    lbl.Position = UDim2.new(0, 16, 0, 12)
    lbl.BackgroundTransparency = 1
    lbl.Text = e.name
    lbl.TextColor3 = T.text
    lbl.Font = Enum.Font.GothamBold
    lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = btn
    local idLbl = Instance.new("TextLabel")
    idLbl.Size = UDim2.new(1, -18, 0, 10)
    idLbl.Position = UDim2.new(0, 16, 0, 32)
    idLbl.BackgroundTransparency = 1
    idLbl.Text = e.id
    idLbl.TextColor3 = T.dim
    idLbl.Font = Enum.Font.Code
    idLbl.TextSize = 8
    idLbl.TextXAlignment = Enum.TextXAlignment.Left
    idLbl.Parent = btn
    btn.MouseEnter:Connect(function()
        tween(btn, 0.18, {BackgroundColor3 = T.cardHover, BackgroundTransparency = 0})
        tween(btnSt, 0.18, {Color = e.color, Transparency = 0.15})
    end)
    btn.MouseLeave:Connect(function()
        tween(btn, 0.18, {BackgroundColor3 = T.card, BackgroundTransparency = 0.15})
        tween(btnSt, 0.18, {Color = T.stroke, Transparency = 0.5})
    end)
    btn.MouseButton1Click:Connect(function()
        State.currentId = e.id
        State.pendingId = e.id
        statusLbl.Text = e.name
        statusLbl.TextColor3 = e.color
        if not isMoving() then
            playEmote("rbxassetid://" .. e.id, e.name)
            State.pendingId = nil
        end
    end)
end

-- TAB SWITCHING
local function showTab(tab)
    State.currentTab = tab
    emotesPage.Visible = (tab == "emotes")
    libraryPage.Visible = (tab == "library")
    spyPage.Visible = (tab == "spy")
    utilsPage.Visible = (tab == "utils")

    local function setTabStyle(btn, active)
        tween(btn, 0.15, {
            BackgroundColor3 = active and T.accent or T.card,
            TextColor3 = active and Color3.fromRGB(10,14,20) or T.subtext
        })
    end
    setTabStyle(emotesTab, tab == "emotes")
    setTabStyle(libraryTab, tab == "library")
    setTabStyle(spyTab, tab == "spy")
    setTabStyle(utilsTab, tab == "utils")
end

emotesTab.MouseButton1Click:Connect(function() showTab("emotes") end)
libraryTab.MouseButton1Click:Connect(function() showTab("library") end)
spyTab.MouseButton1Click:Connect(function() showTab("spy") end)
utilsTab.MouseButton1Click:Connect(function() showTab("utils") end)

-- AIM LOCK
local function getMurdererTarget()
    if not State.aimLock then return nil end
    local mousePos = UserInputService:GetMouseLocation()
    local cam = workspace.CurrentCamera
    local closest, closestDist = nil, State.aimFOV
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= lp and plr.Character then
            local role = State.roleCache[plr.Name]
            if role == "Murderer" then
                local hum = plr.Character:FindFirstChildOfClass("Humanoid")
                local head = plr.Character:FindFirstChild("Head")
                if hum and hum.Health > 0 and head then
                    local screenPos, onScreen = cam:WorldToViewportPoint(head.Position)
                    if onScreen then
                        local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                        if dist < closestDist then
                            closestDist = dist
                            closest = plr
                        end
                    end
                end
            end
        end
    end
    return closest
end

local function predictTargetPos(head)
    if not head then return nil end
    local velocity = head.AssemblyLinearVelocity or Vector3.zero
    local cam = workspace.CurrentCamera
    local distance = (head.Position - cam.CFrame.Position).Magnitude
    local travelTime = math.clamp(distance / 900, 0, 0.3)
    return head.Position + velocity * travelTime
end

track(RunService.RenderStepped:Connect(function()
    if not State.aimLock then return end
    if State.aimUseKey and not UserInputService:IsKeyDown(State.aimKey) then
        State.aimCurrentTarget = nil
        return
    end
    local target = getMurdererTarget()
    if not target or not target.Character then
        State.aimCurrentTarget = nil
        return
    end
    local head = target.Character:FindFirstChild("Head")
    if not head then return end
    State.aimCurrentTarget = target.Name
    local cam = workspace.CurrentCamera
    local currentCFrame = cam.CFrame
    local camPos = currentCFrame.Position
    local targetPos
    if State.aimMode == "PREDICT" then
        targetPos = predictTargetPos(head)
    else
        targetPos = head.Position
    end
    if not targetPos then return end
    local goalCFrame = CFrame.new(camPos, targetPos)
    local jitter = Vector3.new(
        (math.random() - 0.5) * State.aimJitter,
        (math.random() - 0.5) * State.aimJitter,
        (math.random() - 0.5) * State.aimJitter
    )
    goalCFrame = goalCFrame * CFrame.new(jitter)
    if State.aimMode == "SNAP" then
        cam.CFrame = goalCFrame
    else
        cam.CFrame = currentCFrame:Lerp(goalCFrame, State.aimSmoothness)
    end
end))

local function setAuto(on)
    State.autoEmote = on
    if on and State.currentId and not isMoving() then
        playEmote(State.currentId)
    end
end

-- HEADER BUTTONS
closeBtn.MouseEnter:Connect(function() tween(closeBtn, 0.12, {BackgroundColor3 = Color3.fromRGB(80,20,35), TextColor3 = T.red}) end)
closeBtn.MouseLeave:Connect(function() tween(closeBtn, 0.12, {BackgroundColor3 = T.card, TextColor3 = T.subtext}) end)
closeBtn.MouseButton1Click:Connect(function()
    State.open = false
    gui.Enabled = false
    playSound(SOUNDS.toggleOff)
end)

stopBtn.MouseButton1Click:Connect(function()
    stopEmote(true)
    setAuto(false)
end)
stopBtn.MouseEnter:Connect(function() tween(stopBtn, 0.15, {BackgroundColor3 = Color3.fromRGB(90,30,45)}) end)
stopBtn.MouseLeave:Connect(function() tween(stopBtn, 0.15, {BackgroundColor3 = Color3.fromRGB(60,22,35)}) end)

eraseBtn.MouseEnter:Connect(function()
    tween(eraseSt, 0.15, {Transparency = 0.05})
    tween(eraseBtn, 0.15, {BackgroundColor3 = Color3.fromRGB(140,30,55)})
end)
eraseBtn.MouseLeave:Connect(function()
    tween(eraseSt, 0.15, {Transparency = 0.35})
    tween(eraseBtn, 0.15, {BackgroundColor3 = Color3.fromRGB(100,22,38)})
end)
eraseBtn.MouseButton1Click:Connect(function()
    _G.ERASED = true
    stopEmote(true)
    setAuto(false)
    State.chams = false
    State.gunESP = false
    State.autoGrab = false
    State.speed = false
    State.aimLock = false
    State.spyEnabled = false
    clearChams()
    clearGunESP()
    applySpeed()
    for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
    conns = {}
    pcall(function() gui:Destroy() end)
    _G.EmoteMenuCleanup = nil
    error("ERASED", 0)
end)

-- DRAG
local dragging, dragStart, startPos
track(header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = panel.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end))
track(UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local d = input.Position - dragStart
        panel.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
    end
end))

-- KEYBIND
track(UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == HOLD_KEY and os.clock() - LOAD_AT > 0.75 then
        State.open = not State.open
        gui.Enabled = State.open
        playSound(State.open and SOUNDS.toggleOn or SOUNDS.toggleOff)
        return
    end
    if BindWaiting then
        local key = BindWaiting
        BindWaiting = nil
        if input.KeyCode == Enum.KeyCode.Escape then
            local t = toggles[key]
            if t and t.bindBtn then
                t.bindBtn.Text = Binds[key] and Binds[key].Name or "---"
                t.bindBtn.TextColor3 = Binds[key] and T.accent or T.dim
            end
        else
            Binds[key] = input.KeyCode
            local t = toggles[key]
            if t and t.bindBtn then
                t.bindBtn.Text = input.KeyCode.Name
                t.bindBtn.TextColor3 = T.accent
            end
        end
        return
    end
    for key, keyCode in pairs(Binds) do
        if input.KeyCode == keyCode then
            State[key] = not State[key]
            local t = toggles[key]
            if t then
                if t.update then t.update() end
                if t.callback then t.callback(State[key]) end
            end
        end
    end
end))

-- RESPAWN
track(lp.CharacterAdded:Connect(function()
    State.emoteTrack = nil
    State.pendingId = State.currentId
    task.wait(1)
    applySpeed()
    if State.chams then refreshChams() end
    if State.autoEmote and State.currentId then
        task.wait(0.5)
        if not isMoving() then playEmote(State.currentId) end
    end
end))

track(Players.PlayerAdded:Connect(function(plr)
    task.wait(0.5)
    if State.chams then applyChamsForPlayer(plr) end
    track(plr.CharacterAdded:Connect(function()
        task.wait(0.5)
        if State.chams then applyChamsForPlayer(plr) end
    end))
end))

track(Players.PlayerRemoving:Connect(function(plr)
    if chamsInstances[plr.Name] then
        pcall(function() chamsInstances[plr.Name]:Destroy() end)
        chamsInstances[plr.Name] = nil
    end
    clearSpyRow(plr.Name)
end))

-- INIT
do
    local hum = lp.Character and lp.Character:FindFirstChildOfClass("Humanoid")
    if hum then State.defaultSpeed = hum.WalkSpeed end
end

loadLibrary()
refreshLibrary()
updateRoleCache()
refreshChams()
refreshGunESP()
applySpeed()
showTab("emotes")
updateSpyEmpty()

panel.Position = UDim2.new(0, -500, 0.5, -300)
tween(panel, 0.5, {Position = UDim2.new(0, 30, 0.5, -300)}, Enum.EasingStyle.Quint)

-- CLEANUP
_G.EmoteMenuCleanup = function()
    stopEmote(true)
    setAuto(false)
    clearChams()
    clearGunESP()
    for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
    conns = {}
    pcall(function() gui:Destroy() end)
    _G.EmoteMenuCleanup = nil
end

print("[EmoteMenu v16] chargé — Categories + Library + SPY.")
print("[EmoteMenu v16] LeftAlt pour ouvrir/fermer.")
print("[EmoteMenu v16] Filtres : ALL / EMOTE / WALK / IDLE / CUSTOM.")
