--[[
	FunMode — один LocalScript (StarterPlayer → StarterPlayerScripts)
	Підрозділи: Приціл | ТП | ESP | Рух | Світ | Дрони | Налашт.
	Меню: кнопка «МЕНЮ» (або клавіша M на ПК). Вікно тягнеться за заголовок.
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local isTouch = UserInputService.TouchEnabled
local hasMouse = UserInputService.MouseEnabled

-- =====================================================================
-- БЛОК 0. ПРИВ'ЯЗКА ДО ФАН-РЕЖИМУ
-- Якщо REQUIRE_GATE = true, скрипт працює ТІЛЬКИ коли в Workspace є атрибут
-- FunMode = true (ставите його з сервера лише в фан-режимі). В інших режимах
-- меню зникає, а всі функції автоматично вимикаються.
-- =====================================================================
local REQUIRE_GATE = false
local GATE_ATTRIBUTE = "FunMode"

-- =====================================================================
-- БЛОК 1. НАЛАШТУВАННЯ ЗА ЗАМОВЧУВАННЯМ (усе змінюється з меню)
-- =====================================================================
local DEFAULTS = {
	-- Приціл
	Aim = false, AimHold = false, AimPart = 1, Priority = 1,
	Smooth = 0.3, Prediction = 0,
	UseFov = true, FovShow = true, FovRadius = 180, FovThick = 2, FovColor = 3,
	WallCheck = false, IgnoreTeam = false, IncludeNPCs = false,

	-- ТП
	TpAuto = false, TpUseFov = true, TpMode = 1, TpOffset = 4, TpInterval = 2,
	TpDmg = false, TpDmgMin = 15, TpDmgMax = 40, TpDmgCd = 1.5,

	-- ESP
	Esp = false, EspHighlight = true,
	EspFill = 1, EspFillTrans = 0.5, EspOutline = 4, EspOutTrans = 0,
	EspTeamColor = false, EspHealthColor = false, EspAlwaysOnTop = true,
	EspNames = true, EspHealth = true, EspDist = true, EspTextSize = 14,
	EspMaxDist = 2000, EspHideTeam = false,

	-- Рух
	SpeedOn = false, Speed = 32, JumpOn = false, JumpPower = 80,
	InfJump = false, Noclip = false, Fly = false, FlySpeed = 60,
	Spin = false, SpinMode = 1, SpinSpeed = 720,

	-- Світ
	Coins = false, CoinName = "Coin", CoinInterval = 0.3, CoinRange = 300, CoinReturn = true,
	FullBright = false, CamFovOn = false, CamFov = 90,
	GravityOn = false, Gravity = 100,
	Crosshair = false, CrossSize = 12, CrossColor = 2,

	-- Дрони
	DrName = "drone, dron, dr0ne, drohne, uav, fpv, quadcopter, quadrocopter, multicopter, quadro, kamikaze, bomber, "
		.. "дрон, дроны, дрони, беспилотник, безпілотник, бпла, квадрокоптер, мультикоптер, камикадзе, бомбер, бомбардировщик",
	DrExclude = "heli, вертол, гелікоптер",
	DrEsp = false, DrEspColor = 1, DrWarn = false, DrWarnRadius = 120,
	DrAim = false, DrAimRange = 250, DrAutoFire = false,
	DrEvade = false, DrEvadeRadius = 40, DrEvadeDist = 30, DrEvadeCd = 3,

	-- Меню
	Accent = 1, MenuAlpha = 0.04, UiScale = 1, Toasts = true, Quick = isTouch, Debug = false,
	KeyMenu = "M", KeyAim = "F", KeyTp = "T", KeyPanic = "Delete",
}

-- Функції, які вимикає «Вимкнути все» / клавіша паніки
local FEATURES = {
	"Aim", "TpAuto", "TpDmg", "Esp", "SpeedOn", "JumpOn", "InfJump", "Noclip",
	"Fly", "Spin", "Coins", "FullBright", "CamFovOn", "GravityOn", "Crosshair",
	"DrEsp", "DrWarn", "DrAim", "DrAutoFire", "DrEvade",
}

local S = {}
for k, v in DEFAULTS do S[k] = v end

local COLORS = {
	{ "Червоний", Color3.fromRGB(255, 60, 60) },
	{ "Зелений", Color3.fromRGB(0, 220, 110) },
	{ "Блакитний", Color3.fromRGB(60, 200, 255) },
	{ "Білий", Color3.fromRGB(255, 255, 255) },
	{ "Жовтий", Color3.fromRGB(255, 220, 50) },
	{ "Фіолетовий", Color3.fromRGB(170, 90, 255) },
	{ "Помаранчевий", Color3.fromRGB(255, 150, 40) },
	{ "Рожевий", Color3.fromRGB(255, 100, 190) },
}
local COLOR_NAMES = {}
for i, c in COLORS do COLOR_NAMES[i] = c[1] end
local function colorOf(idx) return COLORS[idx][2] end

local ACCENTS = {
	{ "Бірюзовий", Color3.fromRGB(0, 200, 160) },
	{ "Синій", Color3.fromRGB(70, 140, 255) },
	{ "Червоний", Color3.fromRGB(235, 70, 80) },
	{ "Фіолетовий", Color3.fromRGB(150, 100, 255) },
	{ "Помаранчевий", Color3.fromRGB(255, 150, 50) },
	{ "Білий", Color3.fromRGB(235, 235, 240) },
}
local ACCENT_NAMES = {}
for i, c in ACCENTS do ACCENT_NAMES[i] = c[1] end

local AIM_PART_NAMES = { "Голова", "Тулуб", "Корінь" }
local PRIORITY_NAMES = { "Ближче до кола", "Найближчий", "Найменше HP" }
local TP_MODE_NAMES = { "За спину", "Збоку", "Над головою" }
local SPIN_MODE_NAMES = { "Крутилка", "Хаос", "Назад до камери" }

-- =====================================================================
-- БЛОК 2. СТАН І ДОПОМІЖНІ ФУНКЦІЇ
-- =====================================================================
local aimHeld = false
local overMenu = false
local lastAimPoint = Vector2.new(400, 300)
local lastTp, lastTpTry = 0, 0
local spinAngle, spinOn = 0, false
local currentTarget = nil
local npcList, lastNpcScan = {}, 0
local orig = {}          -- збережені значення швидкості/стрибка
local noclipSaved = {}
local listening = nil    -- ключ клавіші, що зараз переназначається
local gateOpen = true

local function getCamera() return workspace.CurrentCamera end
local function getRoot(char) return char and char:FindFirstChild("HumanoidRootPart") end
local function getHum(char) return char and char:FindFirstChildOfClass("Humanoid") end

local function isAlive(char)
	local hum = getHum(char)
	return hum ~= nil and hum.Health > 0
end

local function isTeammate(plr)
	return plr ~= nil and player.Team ~= nil and plr.Team == player.Team
end

local function getAimPart(char)
	local idx = S.AimPart
	local part
	if idx == 1 then
		part = char:FindFirstChild("Head")
	elseif idx == 2 then
		part = char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso")
	end
	return part or char:FindFirstChild("HumanoidRootPart")
end

local function scanNPCs()
	local list = {}
	for _, inst in workspace:GetDescendants() do
		if inst:IsA("Humanoid") then
			local char = inst.Parent
			if char and char ~= player.Character and not Players:GetPlayerFromCharacter(char) then
				list[#list + 1] = char
			end
		end
	end
	npcList = list
end

-- Список цілей: { {char=, plr=}, ... }
local function collect(ignoreTeam)
	local out = {}
	for _, plr in Players:GetPlayers() do
		if plr ~= player and plr.Character and not (ignoreTeam and isTeammate(plr)) then
			out[#out + 1] = { char = plr.Character, plr = plr }
		end
	end
	if S.IncludeNPCs then
		for _, char in npcList do
			if char.Parent then out[#out + 1] = { char = char } end
		end
	end
	return out
end

-- =====================================================================
-- БЛОК 3. ТЕМА ТА ФАБРИКИ ІНТЕРФЕЙСУ
-- =====================================================================
local C_BG = Color3.fromRGB(14, 15, 19)
local C_PANEL = Color3.fromRGB(19, 20, 25)
local C_ROW = Color3.fromRGB(27, 28, 35)
local C_ROW_HI = Color3.fromRGB(35, 37, 46)
local C_LINE = Color3.fromRGB(44, 46, 56)
local C_TEXT = Color3.fromRGB(232, 234, 240)
local C_DIM = Color3.fromRGB(128, 132, 148)
local ACCENT = ACCENTS[S.Accent][2]

local ROW_H = isTouch and 46 or 36
local FOOT_H = 24
local HEAD_H = 40
local SIDE_W = 96

local UI = {}                 -- посилання на основні елементи
local accentRefs = {}         -- елементи, що міняють колір разом з акцентом
local widgets = {}            -- [ключ] = { функції відмальовки }
local toggleLabels = {}       -- [ключ] = назва для сповіщень
local applyVisuals, applyAccent, startCoinLoop, disableAll, notify, droneReport

local function register(key, fn)
	widgets[key] = widgets[key] or {}
	table.insert(widgets[key], fn)
	fn()
end

local function acc(obj, prop)
	table.insert(accentRefs, { obj, prop })
	obj[prop] = ACCENT
end

local function corner(parent, radius)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, radius or 6)
	c.Parent = parent
	return c
end

local function addStroke(parent, color, thickness, transparency)
	local s = Instance.new("UIStroke")
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Color = color
	s.Thickness = thickness or 1
	s.Transparency = transparency or 0
	s.Parent = parent
	return s
end

local function makeText(parent, size, pos, text, textSize, color, font, align)
	local l = Instance.new("TextLabel")
	l.Size = size
	l.Position = pos
	l.BackgroundTransparency = 1
	l.Text = text
	l.TextColor3 = color or C_TEXT
	l.Font = font or Enum.Font.GothamMedium
	l.TextSize = textSize or 13
	l.TextXAlignment = align or Enum.TextXAlignment.Left
	l.TextTruncate = Enum.TextTruncate.AtEnd
	l.Parent = parent
	return l
end

local function setValue(key, value, silent)
	S[key] = value
	for _, r in widgets[key] or {} do r() end
	if key == "Accent" then applyAccent() end
	if applyVisuals then applyVisuals() end
	if key == "Coins" and startCoinLoop then startCoinLoop() end
	if not silent and type(value) == "boolean" and toggleLabels[key] and notify then
		notify(toggleLabels[key], value)
	end
end

-- =====================================================================
-- БЛОК 4. ІНТЕРФЕЙС: основа (оверлей, кнопка, вікно, сповіщення, док)
-- =====================================================================
do
	-- --- Оверлей: коло FOV і хрестик (без відступів — збігається з мишею) ---
	local overlay = Instance.new("ScreenGui")
	overlay.Name = "FunModeOverlay"
	overlay.ResetOnSpawn = false
	overlay.ScreenInsets = Enum.ScreenInsets.None
	overlay.DisplayOrder = 99
	overlay.Parent = playerGui
	UI.overlay = overlay

	local circle = Instance.new("Frame")
	circle.AnchorPoint = Vector2.new(0.5, 0.5)
	circle.BackgroundTransparency = 1
	circle.Visible = false
	circle.Parent = overlay
	corner(circle, 9999)
	UI.circle = circle
	UI.circleStroke = addStroke(circle, Color3.new(1, 1, 1), 2, 0.15)

	local crossH = Instance.new("Frame")
	crossH.AnchorPoint = Vector2.new(0.5, 0.5)
	crossH.Position = UDim2.fromScale(0.5, 0.5)
	crossH.BorderSizePixel = 0
	crossH.Visible = false
	crossH.Parent = overlay
	local crossV = crossH:Clone()
	crossV.Parent = overlay
	UI.crossH, UI.crossV = crossH, crossV

	-- --- Головний GUI (нижче топбару Roblox, поза вирізом екрана) ---
	local gui = Instance.new("ScreenGui")
	gui.Name = "FunModeGui"
	gui.ResetOnSpawn = false
	gui.ScreenInsets = Enum.ScreenInsets.TopbarSafeInsets
	gui.DisplayOrder = 100
	gui.Parent = playerGui
	UI.gui = gui

	-- Кнопка меню
	local menuBtn = Instance.new("TextButton")
	menuBtn.Size = UDim2.fromOffset(isTouch and 84 or 76, isTouch and 38 or 32)
	menuBtn.Position = UDim2.new(0, 12, 0, 8)
	menuBtn.BackgroundColor3 = C_BG
	menuBtn.BackgroundTransparency = 0.1
	menuBtn.AutoButtonColor = false
	menuBtn.Text = "МЕНЮ"
	menuBtn.TextColor3 = C_TEXT
	menuBtn.Font = Enum.Font.GothamBold
	menuBtn.TextSize = 13
	menuBtn.Parent = gui
	corner(menuBtn, 6)
	acc(addStroke(menuBtn, ACCENT, 1.5), "Color")
	UI.menuBtn = menuBtn

	-- Сповіщення (зверху по центру)
	local toasts = Instance.new("Frame")
	toasts.AnchorPoint = Vector2.new(0.5, 0)
	toasts.Position = UDim2.new(0.5, 0, 0, 10)
	toasts.Size = UDim2.fromOffset(260, 0)
	toasts.AutomaticSize = Enum.AutomaticSize.Y
	toasts.BackgroundTransparency = 1
	toasts.Parent = gui
	local tl = Instance.new("UIListLayout")
	tl.Padding = UDim.new(0, 4)
	tl.HorizontalAlignment = Enum.HorizontalAlignment.Center
	tl.SortOrder = Enum.SortOrder.LayoutOrder
	tl.Parent = toasts
	UI.toasts = toasts

	-- Вікно
	local window = Instance.new("Frame")
	window.Name = "Window"
	window.Position = UDim2.new(0, 12, 0, 52)
	window.Size = UDim2.fromOffset(500, 400)
	window.BackgroundColor3 = C_BG
	window.BorderSizePixel = 0
	window.ClipsDescendants = true
	window.Visible = false
	window.Parent = gui
	corner(window, 10)
	addStroke(window, C_LINE, 1)
	UI.window = window

	UI.scale = Instance.new("UIScale")
	UI.scale.Parent = window

	-- Акцентна смужка зверху
	local strip = Instance.new("Frame")
	strip.Size = UDim2.new(1, 0, 0, 2)
	strip.BorderSizePixel = 0
	strip.Parent = window
	acc(strip, "BackgroundColor3")

	-- Заголовок
	local header = Instance.new("Frame")
	header.Position = UDim2.fromOffset(0, 2)
	header.Size = UDim2.new(1, 0, 0, HEAD_H - 2)
	header.BackgroundTransparency = 1
	header.Parent = window
	UI.header = header

	makeText(header, UDim2.new(1, -60, 0, 20), UDim2.fromOffset(14, 5), "FUN MODE", 14, C_TEXT, Enum.Font.GothamBold)
	makeText(header, UDim2.new(1, -60, 0, 14), UDim2.fromOffset(14, 21), "тягни за заголовок, щоб перемістити", 10, C_DIM, Enum.Font.Gotham)

	local closeBtn = Instance.new("TextButton")
	closeBtn.AnchorPoint = Vector2.new(1, 0.5)
	closeBtn.Position = UDim2.new(1, -8, 0.5, 0)
	closeBtn.Size = UDim2.fromOffset(28, 28)
	closeBtn.BackgroundTransparency = 1
	closeBtn.Text = "✕"
	closeBtn.TextColor3 = C_DIM
	closeBtn.Font = Enum.Font.GothamBold
	closeBtn.TextSize = 15
	closeBtn.Parent = header
	UI.closeBtn = closeBtn

	local hLine = Instance.new("Frame")
	hLine.Position = UDim2.fromOffset(0, HEAD_H)
	hLine.Size = UDim2.new(1, 0, 0, 1)
	hLine.BackgroundColor3 = C_LINE
	hLine.BorderSizePixel = 0
	hLine.Parent = window

	-- Бічна панель з підрозділами
	local sidebar = Instance.new("Frame")
	sidebar.Position = UDim2.fromOffset(0, HEAD_H + 1)
	sidebar.Size = UDim2.new(0, SIDE_W, 1, -(HEAD_H + 1 + FOOT_H))
	sidebar.BackgroundColor3 = C_PANEL
	sidebar.BorderSizePixel = 0
	sidebar.Parent = window
	local sl = Instance.new("UIListLayout")
	sl.Padding = UDim.new(0, 2)
	sl.SortOrder = Enum.SortOrder.LayoutOrder
	sl.Parent = sidebar
	local sp = Instance.new("UIPadding")
	sp.PaddingTop = UDim.new(0, 8)
	sp.Parent = sidebar
	UI.sidebar = sidebar

	local sLine = Instance.new("Frame")
	sLine.Position = UDim2.fromOffset(SIDE_W, HEAD_H + 1)
	sLine.Size = UDim2.new(0, 1, 1, -(HEAD_H + 1 + FOOT_H))
	sLine.BackgroundColor3 = C_LINE
	sLine.BorderSizePixel = 0
	sLine.Parent = window

	-- Область вмісту
	local content = Instance.new("Frame")
	content.Position = UDim2.fromOffset(SIDE_W + 1, HEAD_H + 1)
	content.Size = UDim2.new(1, -(SIDE_W + 1), 1, -(HEAD_H + 1 + FOOT_H))
	content.BackgroundTransparency = 1
	content.ClipsDescendants = true
	content.Parent = window
	UI.content = content

	-- Футер
	local footer = Instance.new("Frame")
	footer.AnchorPoint = Vector2.new(0, 1)
	footer.Position = UDim2.fromScale(0, 1)
	footer.Size = UDim2.new(1, 0, 0, FOOT_H)
	footer.BackgroundColor3 = C_PANEL
	footer.BorderSizePixel = 0
	footer.Parent = window
	local fLine = Instance.new("Frame")
	fLine.Size = UDim2.new(1, 0, 0, 1)
	fLine.BackgroundColor3 = C_LINE
	fLine.BorderSizePixel = 0
	fLine.Parent = footer
	UI.footerL = makeText(footer, UDim2.new(0.6, -12, 1, 0), UDim2.fromOffset(12, 0), "", 11, C_DIM, Enum.Font.Gotham)
	UI.footerR = makeText(footer, UDim2.new(0.4, -12, 1, 0), UDim2.new(0.6, 0, 0, 0), "", 11, C_DIM, Enum.Font.Gotham, Enum.TextXAlignment.Right)

	-- Док швидких кнопок (для телефона: праворуч по центру, подалі від стрибка)
	local dock = Instance.new("Frame")
	dock.AnchorPoint = Vector2.new(1, 0.5)
	dock.Position = UDim2.new(1, -10, 0.42, 0)
	dock.Size = UDim2.fromOffset(54, 0)
	dock.AutomaticSize = Enum.AutomaticSize.Y
	dock.BackgroundTransparency = 1
	dock.Visible = false
	dock.Parent = gui
	local dl = Instance.new("UIListLayout")
	dl.Padding = UDim.new(0, 6)
	dl.SortOrder = Enum.SortOrder.LayoutOrder
	dl.Parent = dock
	UI.dock = dock

	-- Не рухати коло FOV, поки курсор над меню
	for _, obj in { window, menuBtn, dock } do
		obj.MouseEnter:Connect(function() overMenu = true end)
		obj.MouseLeave:Connect(function() overMenu = false end)
	end
end

-- --- Сповіщення ---
local function toast(text, color, force)
	if not S.Toasts and not force then return end
	local t = Instance.new("TextLabel")
	t.AutomaticSize = Enum.AutomaticSize.X
	t.Size = UDim2.fromOffset(0, 26)
	t.BackgroundColor3 = C_BG
	t.BackgroundTransparency = 0.1
	t.Text = "  " .. text .. "  "
	t.TextColor3 = color or C_TEXT
	t.Font = Enum.Font.GothamMedium
	t.TextSize = 13
	t.Parent = UI.toasts
	corner(t, 6)
	task.delay(1.8, function()
		TweenService:Create(t, TweenInfo.new(0.3), { BackgroundTransparency = 1, TextTransparency = 1 }):Play()
		task.delay(0.35, function() t:Destroy() end)
	end)
end

notify = function(text, on)
	toast(string.format("%s: %s", text, on and "ВКЛ" or "ВИКЛ"), on and ACCENT or C_DIM)
end

local function openMenu(open)
	if open == nil then open = not UI.window.Visible end
	UI.window.Visible = open
	if not open then overMenu = false end
	if applyVisuals then applyVisuals() end
end

UI.menuBtn.Activated:Connect(function() openMenu() end)
UI.closeBtn.Activated:Connect(function() openMenu(false) end)

-- --- Перетягування вікна за заголовок ---
local dragging
UI.header.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Touch
		or input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = {
			input = input,
			start = Vector2.new(input.Position.X, input.Position.Y),
			pos = UI.window.Position,
		}
	end
end)

-- =====================================================================
-- БЛОК 5. ІНТЕРФЕЙС: підрозділи та віджети
-- =====================================================================
local pages = {}
local currentPage

local function selectTab(page)
	currentPage = page
	for _, p in pages do
		local active = (p == page)
		p.frame.Visible = active
		p.bar.Visible = active
		p.btn.BackgroundTransparency = active and 0 or 1
		p.btn.BackgroundColor3 = C_ROW
		p.btn.TextColor3 = active and C_TEXT or C_DIM
		p.bar.BackgroundColor3 = ACCENT
	end
end

local function addTab(name)
	local frame = Instance.new("ScrollingFrame")
	frame.Size = UDim2.fromScale(1, 1)
	frame.BackgroundTransparency = 1
	frame.BorderSizePixel = 0
	frame.ScrollBarThickness = 3
	frame.ScrollBarImageColor3 = C_LINE
	frame.CanvasSize = UDim2.new()
	frame.AutomaticCanvasSize = Enum.AutomaticSize.Y
	frame.ScrollingDirection = Enum.ScrollingDirection.Y
	frame.Visible = false
	frame.Parent = UI.content

	local pad = Instance.new("UIPadding")
	pad.PaddingTop, pad.PaddingBottom = UDim.new(0, 8), UDim.new(0, 12)
	pad.PaddingLeft, pad.PaddingRight = UDim.new(0, 10), UDim.new(0, 12)
	pad.Parent = frame

	local list = Instance.new("UIListLayout")
	list.Padding = UDim.new(0, 5)
	list.SortOrder = Enum.SortOrder.LayoutOrder
	list.Parent = frame

	local btn = Instance.new("TextButton")
	btn.LayoutOrder = #pages + 1
	btn.Size = UDim2.new(1, 0, 0, isTouch and 40 or 34)
	btn.BackgroundTransparency = 1
	btn.AutoButtonColor = false
	btn.Text = name
	btn.TextColor3 = C_DIM
	btn.Font = Enum.Font.GothamMedium
	btn.TextSize = 13
	btn.TextXAlignment = Enum.TextXAlignment.Left
	btn.Parent = UI.sidebar
	local bp = Instance.new("UIPadding")
	bp.PaddingLeft = UDim.new(0, 16)
	bp.Parent = btn

	local bar = Instance.new("Frame")
	bar.Size = UDim2.new(0, 3, 0.6, 0)
	bar.AnchorPoint = Vector2.new(0, 0.5)
	bar.Position = UDim2.new(0, -16, 0.5, 0)
	bar.BorderSizePixel = 0
	bar.Visible = false
	bar.Parent = btn
	corner(bar, 2)

	local page = { frame = frame, btn = btn, bar = bar, n = 0 }
	table.insert(pages, page)
	btn.Activated:Connect(function() selectTab(page) end)
	return page
end

local function makeRow(page, height, clickable)
	local row = Instance.new(clickable and "TextButton" or "Frame")
	page.n += 1
	row.LayoutOrder = page.n
	row.Size = UDim2.new(1, 0, 0, height)
	row.BackgroundColor3 = C_ROW
	row.BorderSizePixel = 0
	if clickable then
		row.Text = ""
		row.AutoButtonColor = false
	end
	row.Parent = page.frame
	corner(row, 5)
	addStroke(row, C_LINE, 1, 0.3)
	return row
end

local function addSection(page, text)
	page.n += 1
	local holder = Instance.new("Frame")
	holder.LayoutOrder = page.n
	holder.Size = UDim2.new(1, 0, 0, 26)
	holder.BackgroundTransparency = 1
	holder.Parent = page.frame
	makeText(holder, UDim2.new(1, 0, 0, 20), UDim2.fromOffset(2, 6), string.upper(text), 10, C_DIM, Enum.Font.GothamBold)
	local line = Instance.new("Frame")
	line.AnchorPoint = Vector2.new(0, 1)
	line.Position = UDim2.fromScale(0, 1)
	line.Size = UDim2.new(1, 0, 0, 1)
	line.BackgroundColor3 = C_LINE
	line.BorderSizePixel = 0
	line.Parent = holder
end

local function addToggle(page, key, text)
	toggleLabels[key] = text
	local row = makeRow(page, ROW_H, true)
	makeText(row, UDim2.new(1, -66, 1, 0), UDim2.fromOffset(12, 0), text, 13)

	local track = Instance.new("Frame")
	track.AnchorPoint = Vector2.new(1, 0.5)
	track.Size = UDim2.fromOffset(38, 20)
	track.Position = UDim2.new(1, -10, 0.5, 0)
	track.BackgroundColor3 = C_LINE
	track.BorderSizePixel = 0
	track.Parent = row
	corner(track, 10)

	local knob = Instance.new("Frame")
	knob.AnchorPoint = Vector2.new(0, 0.5)
	knob.Size = UDim2.fromOffset(14, 14)
	knob.Position = UDim2.new(0, 3, 0.5, 0)
	knob.BackgroundColor3 = C_DIM
	knob.BorderSizePixel = 0
	knob.Parent = track
	corner(knob, 7)

	local first = true
	register(key, function()
		local on = S[key]
		local tGoal = { BackgroundColor3 = on and ACCENT or C_LINE }
		local kGoal = {
			Position = on and UDim2.new(1, -17, 0.5, 0) or UDim2.new(0, 3, 0.5, 0),
			BackgroundColor3 = on and C_BG or C_DIM,
		}
		if first then
			first = false
			for p, v in tGoal do track[p] = v end
			for p, v in kGoal do knob[p] = v end
		else
			local info = TweenInfo.new(0.15, Enum.EasingStyle.Quad)
			TweenService:Create(track, info, tGoal):Play()
			TweenService:Create(knob, info, kGoal):Play()
		end
	end)

	row.Activated:Connect(function() setValue(key, not S[key]) end)
end

local activeSlider -- { input, update, page }

local function addSlider(page, key, text, minV, maxV, step, fmt)
	local row = makeRow(page, isTouch and 58 or 50, false)
	local label = makeText(row, UDim2.new(1, -100, 0, 20), UDim2.fromOffset(12, 5), text, 13)
	local valueLabel = makeText(row, UDim2.fromOffset(80, 20), UDim2.new(1, -92, 0, 5), "", 12, C_DIM, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
	label.Size = UDim2.new(1, -100, 0, 20)

	local hit = Instance.new("TextButton")
	hit.AnchorPoint = Vector2.new(0.5, 1)
	hit.Size = UDim2.new(1, -24, 0, isTouch and 28 or 22)
	hit.Position = UDim2.new(0.5, 0, 1, -2)
	hit.BackgroundTransparency = 1
	hit.Text = ""
	hit.AutoButtonColor = false
	hit.Parent = row

	local track = Instance.new("Frame")
	track.AnchorPoint = Vector2.new(0, 0.5)
	track.Size = UDim2.new(1, 0, 0, 4)
	track.Position = UDim2.new(0, 0, 0.5, 0)
	track.BackgroundColor3 = C_LINE
	track.BorderSizePixel = 0
	track.Parent = hit
	corner(track, 2)

	local fill = Instance.new("Frame")
	fill.Size = UDim2.new(0, 0, 1, 0)
	fill.BorderSizePixel = 0
	fill.Parent = track
	corner(fill, 2)

	local knob = Instance.new("Frame")
	knob.AnchorPoint = Vector2.new(0.5, 0.5)
	knob.Size = UDim2.fromOffset(isTouch and 16 or 12, isTouch and 16 or 12)
	knob.BackgroundColor3 = C_TEXT
	knob.BorderSizePixel = 0
	knob.Parent = track
	corner(knob, 8)

	register(key, function()
		local rel = math.clamp((S[key] - minV) / (maxV - minV), 0, 1)
		fill.Size = UDim2.new(rel, 0, 1, 0)
		fill.BackgroundColor3 = ACCENT
		knob.Position = UDim2.new(rel, 0, 0.5, 0)
		valueLabel.Text = string.format(fmt, S[key])
	end)

	local function setFromX(x)
		local rel = math.clamp((x - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), 0, 1)
		local value = math.floor((minV + rel * (maxV - minV)) / step + 0.5) * step
		setValue(key, math.clamp(value, minV, maxV))
	end

	hit.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.Touch
			or input.UserInputType == Enum.UserInputType.MouseButton1 then
			activeSlider = { input = input, update = setFromX, page = page }
			page.frame.ScrollingEnabled = false
			setFromX(input.Position.X)
		end
	end)
end

local function addCycle(page, key, text, names)
	local row = makeRow(page, ROW_H, true)
	makeText(row, UDim2.new(0.5, -12, 1, 0), UDim2.fromOffset(12, 0), text, 13)
	local value = makeText(row, UDim2.new(0.5, -12, 1, 0), UDim2.new(0.5, 0, 0, 0), "", 13, ACCENT, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
	register(key, function()
		value.Text = names[S[key]] .. "  ›"
		value.TextColor3 = ACCENT
	end)
	row.Activated:Connect(function()
		setValue(key, S[key] % #names + 1)
	end)
end

local function addButton(page, text, callback)
	local row = makeRow(page, ROW_H, true)
	row.Text = text
	row.TextColor3 = C_TEXT
	row.Font = Enum.Font.GothamBold
	row.TextSize = 13
	row.Activated:Connect(callback)
	row.MouseEnter:Connect(function() row.BackgroundColor3 = C_ROW_HI end)
	row.MouseLeave:Connect(function() row.BackgroundColor3 = C_ROW end)
end

local function addTextBox(page, key, text)
	local row = makeRow(page, ROW_H, false)
	makeText(row, UDim2.new(0.5, -12, 1, 0), UDim2.fromOffset(12, 0), text, 13)
	local tb = Instance.new("TextBox")
	tb.AnchorPoint = Vector2.new(1, 0.5)
	tb.Size = UDim2.new(0.5, -18, 0, 26)
	tb.Position = UDim2.new(1, -10, 0.5, 0)
	tb.BackgroundColor3 = C_BG
	tb.TextColor3 = C_TEXT
	tb.Font = Enum.Font.Gotham
	tb.TextSize = 13
	tb.ClearTextOnFocus = false
	tb.Parent = row
	corner(tb, 5)
	addStroke(tb, C_LINE, 1)
	register(key, function() tb.Text = tostring(S[key]) end)
	tb.FocusLost:Connect(function() setValue(key, tb.Text) end)
end

-- Багаторядкове поле (для довгих списків назв)
local function addTextArea(page, key, text, height)
	local row = makeRow(page, height or 96, false)
	makeText(row, UDim2.new(1, -24, 0, 20), UDim2.fromOffset(12, 5), text, 13)
	local tb = Instance.new("TextBox")
	tb.Position = UDim2.fromOffset(10, 28)
	tb.Size = UDim2.new(1, -20, 1, -36)
	tb.BackgroundColor3 = C_BG
	tb.TextColor3 = C_TEXT
	tb.Font = Enum.Font.Code
	tb.TextSize = 12
	tb.TextWrapped = true
	tb.MultiLine = true
	tb.TextXAlignment = Enum.TextXAlignment.Left
	tb.TextYAlignment = Enum.TextYAlignment.Top
	tb.ClearTextOnFocus = false
	tb.Parent = row
	corner(tb, 5)
	addStroke(tb, C_LINE, 1)
	mkPad = Instance.new("UIPadding")
	mkPad.PaddingLeft, mkPad.PaddingRight = UDim.new(0, 6), UDim.new(0, 6)
	mkPad.PaddingTop, mkPad.PaddingBottom = UDim.new(0, 4), UDim.new(0, 4)
	mkPad.Parent = tb
	register(key, function() tb.Text = tostring(S[key]) end)
	tb.FocusLost:Connect(function() setValue(key, tb.Text) end)
end

local function addKeybind(page, key, text)
	local row = makeRow(page, ROW_H, true)
	makeText(row, UDim2.new(1, -110, 1, 0), UDim2.fromOffset(12, 0), text, 13)
	local keyLabel = Instance.new("TextLabel")
	keyLabel.AnchorPoint = Vector2.new(1, 0.5)
	keyLabel.Size = UDim2.fromOffset(86, 24)
	keyLabel.Position = UDim2.new(1, -10, 0.5, 0)
	keyLabel.BackgroundColor3 = C_BG
	keyLabel.TextColor3 = C_TEXT
	keyLabel.Font = Enum.Font.GothamBold
	keyLabel.TextSize = 12
	keyLabel.Parent = row
	corner(keyLabel, 5)
	addStroke(keyLabel, C_LINE, 1)
	register(key, function()
		keyLabel.Text = (listening == key) and "натисни..." or S[key]
	end)
	row.Activated:Connect(function()
		listening = key
		for _, r in widgets[key] do r() end
	end)
end

-- --- Швидкі кнопки (док) ---
local function addDock(text, key, onTap)
	local b = Instance.new("TextButton")
	b.Size = UDim2.fromOffset(54, 40)
	b.BackgroundColor3 = C_BG
	b.BackgroundTransparency = 0.12
	b.AutoButtonColor = false
	b.Text = text
	b.TextColor3 = C_TEXT
	b.Font = Enum.Font.GothamBold
	b.TextSize = 12
	b.Parent = UI.dock
	corner(b, 6)
	local s = addStroke(b, C_LINE, 1.5)
	b.Activated:Connect(onTap)
	if key then
		register(key, function()
			s.Color = S[key] and ACCENT or C_LINE
			b.TextColor3 = S[key] and ACCENT or C_TEXT
		end)
	end
end

-- =====================================================================
-- БЛОК 6. ВІЗУАЛЬНІ ЕФЕКТИ (коло, вікно, хрестик, світло, гравітація)
-- =====================================================================
local lightingSaved, savedGravity

local function countActive()
	local n = 0
	for _, key in FEATURES do
		if S[key] then n += 1 end
	end
	return n
end

local function updateFooter()
	local txt = "Активно: " .. countActive()
	if S.Debug then
		local name = currentTarget and (currentTarget.plr and currentTarget.plr.Name or currentTarget.char.Name) or "—"
		txt = txt .. "  ·  ціль: " .. name
	end
	UI.footerL.Text = txt
	UI.footerR.Text = isTouch and "" or (S.KeyMenu .. " — меню")
end

applyAccent = function()
	ACCENT = ACCENTS[S.Accent][2]
	for _, ref in accentRefs do ref[1][ref[2]] = ACCENT end
	if currentPage then selectTab(currentPage) end
	for _, list in widgets do
		for _, r in list do r() end
	end
end

applyVisuals = function()
	-- Коло FOV
	UI.circle.Size = UDim2.fromOffset(S.FovRadius * 2, S.FovRadius * 2)
	UI.circleStroke.Color = colorOf(S.FovColor)
	UI.circleStroke.Thickness = S.FovThick
	local fovNeeded = (S.Aim and S.UseFov) or (S.TpAuto and S.TpUseFov) or UI.window.Visible
	UI.circle.Visible = S.FovShow and fovNeeded and (S.UseFov or S.TpUseFov)

	-- Вікно
	UI.window.BackgroundTransparency = S.MenuAlpha
	UI.scale.Scale = S.UiScale
	local cam = getCamera()
	local vp = cam and cam.ViewportSize or Vector2.new(800, 600)
	UI.window.Size = UDim2.fromOffset(
		math.clamp((vp.X - 24) / S.UiScale, 300, 520),
		math.clamp((vp.Y - 70) / S.UiScale, 240, 440)
	)

	-- Док швидких кнопок
	UI.dock.Visible = S.Quick and gateOpen

	-- Хрестик
	UI.crossH.Visible, UI.crossV.Visible = S.Crosshair, S.Crosshair
	UI.crossH.Size = UDim2.fromOffset(S.CrossSize * 2, 2)
	UI.crossV.Size = UDim2.fromOffset(2, S.CrossSize * 2)
	UI.crossH.BackgroundColor3 = colorOf(S.CrossColor)
	UI.crossV.BackgroundColor3 = colorOf(S.CrossColor)

	-- Повна яскравість
	if S.FullBright then
		if not lightingSaved then
			lightingSaved = {
				Brightness = Lighting.Brightness, ClockTime = Lighting.ClockTime,
				FogEnd = Lighting.FogEnd, GlobalShadows = Lighting.GlobalShadows,
				Ambient = Lighting.Ambient, OutdoorAmbient = Lighting.OutdoorAmbient,
			}
		end
		Lighting.Brightness = 2
		Lighting.ClockTime = 14
		Lighting.FogEnd = 1e6
		Lighting.GlobalShadows = false
		Lighting.Ambient = Color3.fromRGB(178, 178, 178)
		Lighting.OutdoorAmbient = Color3.fromRGB(178, 178, 178)
	elseif lightingSaved then
		for prop, val in lightingSaved do Lighting[prop] = val end
		lightingSaved = nil
	end

	-- Гравітація
	if S.GravityOn then
		if not savedGravity then savedGravity = workspace.Gravity end
		workspace.Gravity = S.Gravity
	elseif savedGravity then
		workspace.Gravity = savedGravity
		savedGravity = nil
	end

	updateFooter()
end

-- =====================================================================
-- БЛОК 7. ПОШУК ЦІЛІ
-- =====================================================================
-- ПК: центр кола = курсор (поки курсор не над меню). Телефон: центр екрана.
local function getAimPoint()
	local cam = getCamera()
	if isTouch and not hasMouse then
		return cam.ViewportSize / 2
	end
	if not overMenu then
		lastAimPoint = UserInputService:GetMouseLocation()
	end
	return lastAimPoint
end

local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Exclude

-- useFov = true: тільки цілі всередині кола. Повертає { char=, plr= } або nil
local function getTarget(useFov)
	local cam = getCamera()
	if not cam then return nil end

	local point = getAimPoint()
	local camPos = cam.CFrame.Position
	local myChar = player.Character
	rayParams.FilterDescendantsInstances = myChar and { myChar } or {}

	local best, bestScore = nil, math.huge
	for _, t in collect(S.IgnoreTeam) do
		local char = t.char
		local hum, part = getHum(char), getAimPart(char)
		if hum and part and hum.Health > 0 then
			local sp, onScreen = cam:WorldToViewportPoint(part.Position)
			local d2 = (Vector2.new(sp.X, sp.Y) - point).Magnitude
			local dist = (part.Position - camPos).Magnitude

			local pass = true
			if useFov then pass = onScreen and d2 <= S.FovRadius end
			if pass and S.WallCheck then
				local hit = workspace:Raycast(camPos, part.Position - camPos, rayParams)
				pass = hit == nil or hit.Instance:IsDescendantOf(char)
			end

			if pass then
				local score
				if S.Priority == 1 then
					score = onScreen and d2 or (1e6 + dist)
				elseif S.Priority == 2 then
					score = dist
				else
					score = hum.Health
				end
				if score < bestScore then
					best, bestScore = t, score
				end
			end
		end
	end
	return best
end

-- =====================================================================
-- БЛОК 8. ТЕЛЕПОРТИ
-- =====================================================================
local function teleportToTarget()
	local myChar = player.Character
	local myRoot = getRoot(myChar)
	if not (isAlive(myChar) and myRoot) then return false end

	local t = getTarget(S.TpUseFov)
	local tRoot = t and getRoot(t.char)
	if not tRoot then return false end

	lastTp = os.clock()

	local offset
	if S.TpMode == 1 then
		offset = -tRoot.CFrame.LookVector * S.TpOffset
	elseif S.TpMode == 2 then
		offset = tRoot.CFrame.RightVector * S.TpOffset
	else
		offset = Vector3.new(0, S.TpOffset, 0)
	end

	-- Якщо на шляху стіна, зупиняємось перед нею
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { myChar, t.char }
	local pos = tRoot.Position + offset
	local hit = workspace:Raycast(tRoot.Position, offset, params)
	if hit then pos = hit.Position + hit.Normal * 2 end

	local cf
	if S.TpMode == 1 then
		cf = CFrame.new(pos) * tRoot.CFrame.Rotation
	else
		local look = Vector3.new(tRoot.Position.X, pos.Y, tRoot.Position.Z)
		cf = (look - pos).Magnitude > 0.05 and CFrame.lookAt(pos, look) or CFrame.new(pos)
	end

	myChar:PivotTo(cf)
	myRoot.AssemblyLinearVelocity = Vector3.zero
	return true
end

local function randomTeleport(char)
	local root = getRoot(char)
	if not root then return end

	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { char }

	local minD = math.min(S.TpDmgMin, S.TpDmgMax)
	local maxD = math.max(S.TpDmgMin, S.TpDmgMax)

	for _ = 1, 8 do
		local angle = math.rad(math.random(0, 359))
		local dist = math.random(minD, maxD)
		local origin = root.Position + Vector3.new(math.cos(angle) * dist, 50, math.sin(angle) * dist)
		local hit = workspace:Raycast(origin, Vector3.new(0, -200, 0), params)
		if hit and hit.Normal.Y > 0.7 then
			char:PivotTo(CFrame.new(hit.Position + Vector3.new(0, 4, 0)))
			root.AssemblyLinearVelocity = Vector3.zero
			return
		end
	end
end

local lastDmgTp = 0
local function hookCharacter(char)
	local hum = char:WaitForChild("Humanoid", 10)
	if not hum then return end
	local last = hum.Health
	hum.HealthChanged:Connect(function(new)
		if S.TpDmg and new < last and new > 0 and os.clock() - lastDmgTp >= S.TpDmgCd then
			lastDmgTp = os.clock()
			randomTeleport(char)
		end
		last = new
	end)
end

player.CharacterAdded:Connect(function(char)
	orig = {}
	hookCharacter(char)
end)
if player.Character then task.spawn(hookCharacter, player.Character) end

-- =====================================================================
-- БЛОК 9. ESP (Highlight + підписи)
-- =====================================================================
local espObjs = {} -- [char] = { hl, bb, name, info, bar, fill }
local updateEsp

do
	local espFolder = Instance.new("Folder")
	espFolder.Name = "FunModeESP"
	espFolder.Parent = playerGui

	local function createEsp(char)
		local hl = Instance.new("Highlight")
		hl.Name = "FunModeHighlight"
		hl.Adornee = char
		hl.Parent = char

		local bb = Instance.new("BillboardGui")
		bb.Name = "FunModeTag"
		bb.AlwaysOnTop = true
		bb.Size = UDim2.fromOffset(200, 44)
		bb.StudsOffset = Vector3.new(0, 3.6, 0)
		bb.Adornee = getRoot(char)
		bb.Parent = espFolder

		local function line(y)
			local l = Instance.new("TextLabel")
			l.Size = UDim2.new(1, 0, 0, 16)
			l.Position = UDim2.fromOffset(0, y)
			l.BackgroundTransparency = 1
			l.TextColor3 = Color3.new(1, 1, 1)
			l.TextStrokeTransparency = 0.3
			l.Font = Enum.Font.GothamBold
			l.Parent = bb
			return l
		end

		local bar = Instance.new("Frame")
		bar.AnchorPoint = Vector2.new(0.5, 0)
		bar.Position = UDim2.new(0.5, 0, 0, 34)
		bar.Size = UDim2.fromOffset(90, 5)
		bar.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
		bar.BorderSizePixel = 0
		bar.Parent = bb

		local fill = Instance.new("Frame")
		fill.Size = UDim2.fromScale(1, 1)
		fill.BorderSizePixel = 0
		fill.Parent = bar

		return { hl = hl, bb = bb, name = line(0), info = line(16), bar = bar, fill = fill }
	end

	local function destroyEsp(char)
		local e = espObjs[char]
		if e then
			e.hl:Destroy()
			e.bb:Destroy()
			espObjs[char] = nil
		end
	end

	updateEsp = function()
		local cam = getCamera()
		if not cam then return end
		if not S.Esp and next(espObjs) == nil then return end

		local camPos = cam.CFrame.Position
		local seen = {}

		if S.Esp then
			for _, t in collect(false) do
				local char, plr = t.char, t.plr
				local hum, root = getHum(char), getRoot(char)
				if hum and root and hum.Health > 0 then
					local dist = (root.Position - camPos).Magnitude
					local hide = (S.EspHideTeam and isTeammate(plr))
						or (S.EspMaxDist < 5000 and dist > S.EspMaxDist)

					if not hide then
						seen[char] = true
						local e = espObjs[char]
						if not e then
							e = createEsp(char)
							espObjs[char] = e
						end

						local frac = math.clamp(hum.Health / math.max(hum.MaxHealth, 1), 0, 1)
						local fillColor = colorOf(S.EspFill)
						local outColor = colorOf(S.EspOutline)
						if S.EspTeamColor and plr and plr.Team then
							fillColor = plr.TeamColor.Color
							outColor = plr.TeamColor.Color
						end
						if S.EspHealthColor then
							fillColor = Color3.fromRGB(255, 60, 60):Lerp(Color3.fromRGB(60, 255, 90), frac)
						end

						e.hl.Enabled = S.EspHighlight
						e.hl.FillColor = fillColor
						e.hl.OutlineColor = outColor
						e.hl.FillTransparency = S.EspFillTrans
						e.hl.OutlineTransparency = S.EspOutTrans
						e.hl.DepthMode = S.EspAlwaysOnTop
							and Enum.HighlightDepthMode.AlwaysOnTop
							or Enum.HighlightDepthMode.Occluded

						e.bb.Enabled = S.EspNames or S.EspHealth or S.EspDist
						e.bb.Adornee = root

						e.name.Visible = S.EspNames
						e.name.TextSize = S.EspTextSize
						e.name.Text = plr and plr.DisplayName or char.Name

						local parts = {}
						if S.EspHealth then
							parts[#parts + 1] = string.format("%d HP", math.floor(hum.Health + 0.5))
						end
						if S.EspDist then
							parts[#parts + 1] = string.format("%d ст", math.floor(dist + 0.5))
						end
						e.info.Visible = #parts > 0
						e.info.TextSize = S.EspTextSize - 1
						e.info.Text = table.concat(parts, "  |  ")

						e.bar.Visible = S.EspHealth
						e.fill.Size = UDim2.fromScale(frac, 1)
						e.fill.BackgroundColor3 = Color3.fromRGB(255, 60, 60):Lerp(Color3.fromRGB(60, 255, 90), frac)
					end
				end
			end
		end

		for char in espObjs do
			if not seen[char] then destroyEsp(char) end
		end
	end
end

-- =====================================================================
-- БЛОК 10. ЗБІР МОНЕТ
-- =====================================================================
do
	local coinThread = false

	startCoinLoop = function()
		if coinThread or not S.Coins then return end
		coinThread = true
		task.spawn(function()
			local cache, lastScan = {}, 0
			while S.Coins do
				local char = player.Character
				local root = getRoot(char)
				local pattern = string.lower(S.CoinName)

				if root and isAlive(char) and pattern ~= "" then
					-- Сканування карти раз на 3 секунди
					if os.clock() - lastScan > 3 then
						lastScan = os.clock()
						cache = {}
						for _, d in workspace:GetDescendants() do
							if d:IsA("BasePart") and string.find(string.lower(d.Name), pattern, 1, true) then
								local model = d:FindFirstAncestorOfClass("Model")
								if not (model and Players:GetPlayerFromCharacter(model)) then
									cache[#cache + 1] = d
								end
							end
						end
					end

					-- Найближча монета в радіусі
					local best, bestDist = nil, S.CoinRange
					for _, coin in cache do
						if coin.Parent then
							local d = (coin.Position - root.Position).Magnitude
							if d <= bestDist then best, bestDist = coin, d end
						end
					end

					if best then
						local origin = root.CFrame
						char:PivotTo(CFrame.new(best.Position))
						RunService.Heartbeat:Wait()
						if S.CoinReturn and char.Parent then
							char:PivotTo(origin)
						end
					end
				end
				task.wait(S.CoinInterval)
			end
			coinThread = false
		end)
	end
end

-- =====================================================================
-- БЛОК 11. ПОБУДОВА МЕНЮ
-- =====================================================================
disableAll = function()
	for _, key in FEATURES do
		if S[key] then setValue(key, false, true) end
	end
end

local function resetAll()
	for k, v in DEFAULTS do setValue(k, v, true) end
end

do
	-- ---------- Приціл ----------
	local p = addTab("Приціл")
	addSection(p, "Автоприціл")
	addToggle(p, "Aim", "Автоприціл")
	addToggle(p, "AimHold", "Тільки при утриманні ПКМ (ПК)")
	addCycle(p, "AimPart", "Куди цілитись", AIM_PART_NAMES)
	addCycle(p, "Priority", "Пріоритет цілі", PRIORITY_NAMES)
	addSlider(p, "Smooth", "Різкість наведення", 0.05, 1, 0.05, "%.2f")
	addSlider(p, "Prediction", "Передбачення руху", 0, 0.5, 0.01, "%.2f с")
	addSection(p, "Коло FOV")
	addToggle(p, "UseFov", "Обмежувати колом FOV")
	addToggle(p, "FovShow", "Показувати коло")
	addSlider(p, "FovRadius", "Радіус кола", 30, 800, 10, "%d px")
	addSlider(p, "FovThick", "Товщина лінії", 1, 6, 1, "%d")
	addCycle(p, "FovColor", "Колір кола", COLOR_NAMES)
	addSection(p, "Цілі")
	addToggle(p, "WallCheck", "Не цілитись крізь стіни")
	addToggle(p, "IgnoreTeam", "Ігнорувати свою команду")
	addToggle(p, "IncludeNPCs", "Включати NPC / манекени")

	-- ---------- ТП ----------
	local t = addTab("ТП")
	addSection(t, "Телепорт до цілі")
	addToggle(t, "TpAuto", "Авто-ТП до цілі")
	addButton(t, "ТП до цілі зараз", teleportToTarget)
	addCycle(t, "TpMode", "Позиція", TP_MODE_NAMES)
	addSlider(t, "TpOffset", "Відстань від цілі", 1, 20, 1, "%d ст")
	addSlider(t, "TpInterval", "Інтервал авто-ТП", 0.5, 10, 0.5, "%.1f с")
	addToggle(t, "TpUseFov", "Лише ціль у колі FOV")
	addSection(t, "ТП при отриманні урону")
	addToggle(t, "TpDmg", "Рандомний ТП при уроні")
	addSlider(t, "TpDmgMin", "Мін. дистанція", 5, 100, 5, "%d ст")
	addSlider(t, "TpDmgMax", "Макс. дистанція", 10, 200, 5, "%d ст")
	addSlider(t, "TpDmgCd", "Кулдаун", 0.5, 10, 0.5, "%.1f с")

	-- ---------- ESP ----------
	local e = addTab("ESP")
	addSection(e, "Загальне")
	addToggle(e, "Esp", "Увімкнути ESP")
	addToggle(e, "EspHighlight", "Підсвітка тіла")
	addToggle(e, "EspAlwaysOnTop", "Видно крізь стіни")
	addToggle(e, "EspHideTeam", "Ховати свою команду")
	addSlider(e, "EspMaxDist", "Дальність (5000 = без ліміту)", 100, 5000, 100, "%d")
	addSection(e, "Кольори")
	addCycle(e, "EspFill", "Колір заливки", COLOR_NAMES)
	addSlider(e, "EspFillTrans", "Прозорість заливки", 0, 1, 0.05, "%.2f")
	addCycle(e, "EspOutline", "Колір контуру", COLOR_NAMES)
	addSlider(e, "EspOutTrans", "Прозорість контуру", 0, 1, 0.05, "%.2f")
	addToggle(e, "EspTeamColor", "Колір за командою")
	addToggle(e, "EspHealthColor", "Колір заливки за HP")
	addSection(e, "Підписи")
	addToggle(e, "EspNames", "Імена")
	addToggle(e, "EspHealth", "HP і смужка здоров'я")
	addToggle(e, "EspDist", "Дистанція")
	addSlider(e, "EspTextSize", "Розмір тексту", 10, 24, 1, "%d")

	-- ---------- Рух ----------
	local m = addTab("Рух")
	addSection(m, "Швидкість і стрибок")
	addToggle(m, "SpeedOn", "Власна швидкість")
	addSlider(m, "Speed", "Швидкість бігу", 16, 200, 2, "%d")
	addToggle(m, "JumpOn", "Власна сила стрибка")
	addSlider(m, "JumpPower", "Сила стрибка", 50, 300, 5, "%d")
	addToggle(m, "InfJump", "Нескінченний стрибок")
	addSection(m, "Пересування")
	addToggle(m, "Noclip", "Noclip (крізь стіни)")
	addToggle(m, "Fly", "Політ")
	addSlider(m, "FlySpeed", "Швидкість польоту", 10, 200, 5, "%d")
	addSection(m, "Антиприціл")
	addToggle(m, "Spin", "Антиприціл")
	addCycle(m, "SpinMode", "Режим", SPIN_MODE_NAMES)
	addSlider(m, "SpinSpeed", "Швидкість крутилки", 60, 3000, 60, "%d °/с")

	-- ---------- Світ ----------
	local w = addTab("Світ")
	addSection(w, "Збір монет")
	addToggle(w, "Coins", "Авто-збір монет")
	addTextBox(w, "CoinName", "Назва об'єкта")
	addSlider(w, "CoinInterval", "Інтервал", 0.1, 2, 0.1, "%.1f с")
	addSlider(w, "CoinRange", "Радіус пошуку", 20, 2000, 20, "%d ст")
	addToggle(w, "CoinReturn", "Повертатись назад")
	addSection(w, "Візуал")
	addToggle(w, "FullBright", "Повна яскравість")
	addToggle(w, "CamFovOn", "Власний FOV камери")
	addSlider(w, "CamFov", "FOV камери", 40, 120, 1, "%d°")
	addToggle(w, "GravityOn", "Власна гравітація")
	addSlider(w, "Gravity", "Гравітація", 10, 400, 10, "%d")
	addToggle(w, "Crosshair", "Хрестик у центрі")
	addSlider(w, "CrossSize", "Розмір хрестика", 4, 40, 1, "%d")
	addCycle(w, "CrossColor", "Колір хрестика", COLOR_NAMES)

	-- ---------- Дрони ----------
	local dr = addTab("Дрони")
	addSection(dr, "Пошук дронів")
	addTextArea(dr, "DrName", "Назви дронів (через кому)", 118)
	addTextArea(dr, "DrExclude", "Виключити (через кому)", 72)
	addButton(dr, "Перевірити: скільки знайдено", function() droneReport() end)
	addSection(dr, "Виявлення")
	addToggle(dr, "DrEsp", "Підсвітка дронів")
	addCycle(dr, "DrEspColor", "Колір підсвітки", COLOR_NAMES)
	addToggle(dr, "DrWarn", "Попередження про дрон")
	addSlider(dr, "DrWarnRadius", "Радіус попередження", 30, 400, 10, "%d ст")
	addSection(dr, "Збивання")
	addToggle(dr, "DrAim", "Автоприціл на дрони")
	addSlider(dr, "DrAimRange", "Дальність прицілу", 30, 800, 10, "%d ст")
	addToggle(dr, "DrAutoFire", "Автовогонь по дрону")
	addSection(dr, "Ухилення")
	addToggle(dr, "DrEvade", "Авто-ухилення від дрона")
	addSlider(dr, "DrEvadeRadius", "Радіус спрацювання", 10, 120, 5, "%d ст")
	addSlider(dr, "DrEvadeDist", "Відстань ухилення", 10, 80, 5, "%d ст")
	addSlider(dr, "DrEvadeCd", "Кулдаун ухилення", 1, 10, 0.5, "%.1f с")

	-- ---------- Налаштування ----------
	local s = addTab("Налашт.")
	addSection(s, "Вигляд меню")
	addCycle(s, "Accent", "Акцентний колір", ACCENT_NAMES)
	addSlider(s, "MenuAlpha", "Прозорість меню", 0, 0.8, 0.05, "%.2f")
	addSlider(s, "UiScale", "Масштаб меню", 0.7, 1.4, 0.05, "%.2f")
	addToggle(s, "Toasts", "Сповіщення про вмик/вимк")
	addToggle(s, "Quick", "Швидкі кнопки на екрані")
	addToggle(s, "Debug", "Діагностика (ціль у футері)")
	addSection(s, "Клавіші (ПК), Backspace = прибрати")
	addKeybind(s, "KeyMenu", "Відкрити меню")
	addKeybind(s, "KeyAim", "Вмик/вимк автоприціл")
	addKeybind(s, "KeyTp", "ТП до цілі")
	addKeybind(s, "KeyPanic", "Вимкнути все")
	addSection(s, "Скидання")
	addButton(s, "Вимкнути всі функції", disableAll)
	addButton(s, "Скинути всі налаштування", resetAll)

	selectTab(pages[1])

	-- ---------- Швидкі кнопки ----------
	addDock("AIM", "Aim", function() setValue("Aim", not S.Aim) end)
	addDock("TP", nil, function() teleportToTarget() end)
	addDock("ESP", "Esp", function() setValue("Esp", not S.Esp) end)
	addDock("SPIN", "Spin", function() setValue("Spin", not S.Spin) end)
end

applyAccent()
applyVisuals()

-- =====================================================================
-- БЛОК 12. ПРИВ'ЯЗКА ДО ФАН-РЕЖИМУ (gate)
-- =====================================================================
local function updateGate()
	local open = (not REQUIRE_GATE) or workspace:GetAttribute(GATE_ATTRIBUTE) == true
	if open == gateOpen then return end
	gateOpen = open
	UI.gui.Enabled = open
	UI.overlay.Enabled = open
	if not open then
		openMenu(false)
		disableAll()
	end
	applyVisuals()
end
workspace:GetAttributeChangedSignal(GATE_ATTRIBUTE):Connect(updateGate)
updateGate()

-- =====================================================================
-- БЛОК 13. ГОЛОВНІ ЦИКЛИ
-- =====================================================================
local fovSaved
local espTimer = 0

-- Камера: наведення, FOV камери, позиція кола (після стандартної камери Roblox)
RunService:BindToRenderStep("FunModeRender", Enum.RenderPriority.Camera.Value + 1, function()
	local cam = getCamera()
	if not cam then return end

	local p = getAimPoint()
	UI.circle.Position = UDim2.fromOffset(p.X, p.Y)

	if S.CamFovOn then
		if not fovSaved then fovSaved = cam.FieldOfView end
		cam.FieldOfView = S.CamFov
	elseif fovSaved then
		cam.FieldOfView = fovSaved
		fovSaved = nil
	end

	local active = S.Aim and (not S.AimHold or not hasMouse or aimHeld)
	if active or S.Debug then
		currentTarget = getTarget(S.UseFov)
	else
		currentTarget = nil
	end

	if active and currentTarget then
		local part = getAimPart(currentTarget.char)
		if part then
			local pos = part.Position + part.AssemblyLinearVelocity * S.Prediction
			cam.CFrame = cam.CFrame:Lerp(CFrame.lookAt(cam.CFrame.Position, pos), S.Smooth)
		end
	end
end)

-- Рух, ТП, антиприціл, ESP
RunService.Heartbeat:Connect(function(dt)
	local char = player.Character
	local root, hum = getRoot(char), getHum(char)

	if S.IncludeNPCs and os.clock() - lastNpcScan > 2 then
		lastNpcScan = os.clock()
		scanNPCs()
	end

	if root and hum then
		-- Швидкість
		if S.SpeedOn then
			if orig.walk == nil then orig.walk = hum.WalkSpeed end
			hum.WalkSpeed = S.Speed
		elseif orig.walk ~= nil then
			hum.WalkSpeed = orig.walk
			orig.walk = nil
		end

		-- Стрибок
		if S.JumpOn then
			if orig.jump == nil then
				orig.jump = hum.JumpPower
				orig.useJP = hum.UseJumpPower
			end
			hum.UseJumpPower = true
			hum.JumpPower = S.JumpPower
		elseif orig.jump ~= nil then
			hum.UseJumpPower = orig.useJP
			hum.JumpPower = orig.jump
			orig.jump = nil
		end

		-- Авто-ТП
		if S.TpAuto and os.clock() - lastTp >= S.TpInterval and os.clock() - lastTpTry >= 0.1 then
			lastTpTry = os.clock()
			teleportToTarget()
		end

		-- Політ (джойстик/WASD + погляд камери вгору/вниз = набір висоти)
		if S.Fly then
			local cam = getCamera()
			local move = hum.MoveDirection
			local vel = Vector3.zero
			if cam and move.Magnitude > 0.01 then
				local look = cam.CFrame.LookVector
				local flat = Vector3.new(look.X, 0, look.Z)
				local fwd = flat.Magnitude > 0.01 and move:Dot(flat.Unit) or 0
				vel = move * S.FlySpeed + Vector3.new(0, look.Y * fwd * S.FlySpeed, 0)
			end
			if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
				vel += Vector3.new(0, S.FlySpeed, 0)
			end
			if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift)
				or UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
				vel -= Vector3.new(0, S.FlySpeed, 0)
			end
			root.AssemblyLinearVelocity = vel
		end

		-- Антиприціл
		if S.Spin then
			hum.AutoRotate = false
			spinOn = true
			local pos = root.Position
			if S.SpinMode == 1 then
				spinAngle = (spinAngle + S.SpinSpeed * dt) % 360
				root.CFrame = CFrame.new(pos) * CFrame.Angles(0, math.rad(spinAngle), 0)
			elseif S.SpinMode == 2 then
				root.CFrame = CFrame.new(pos) * CFrame.Angles(0, math.rad(math.random(0, 359)), 0)
			else
				local cam = getCamera()
				local look = cam and cam.CFrame.LookVector or Vector3.zAxis
				local flat = Vector3.new(look.X, 0, look.Z)
				if flat.Magnitude > 0.01 then
					root.CFrame = CFrame.lookAt(pos, pos - flat)
				end
			end
		elseif spinOn then
			hum.AutoRotate = true
			spinOn = false
		end
	end

	-- ESP (10 разів на секунду)
	espTimer += dt
	if espTimer >= 0.1 then
		espTimer = 0
		updateEsp()
		if S.Debug then updateFooter() end
	end
end)

-- Noclip
RunService.Stepped:Connect(function()
	local char = player.Character
	if S.Noclip and char then
		for _, p in char:GetDescendants() do
			if p:IsA("BasePart") and p.CanCollide then
				noclipSaved[p] = true
				p.CanCollide = false
			end
		end
	elseif next(noclipSaved) ~= nil then
		for p in noclipSaved do
			if p.Parent then p.CanCollide = true end
		end
		table.clear(noclipSaved)
	end
end)

-- Нескінченний стрибок
UserInputService.JumpRequest:Connect(function()
	if S.InfJump then
		local hum = getHum(player.Character)
		if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
	end
end)

-- =====================================================================
-- БЛОК 14. ВВІД
-- =====================================================================
local ACTIONS = {
	KeyMenu = function() openMenu() end,
	KeyAim = function() setValue("Aim", not S.Aim) end,
	KeyTp = function() teleportToTarget() end,
	KeyPanic = disableAll,
}

UserInputService.InputBegan:Connect(function(input, processed)
	-- Режим переназначення клавіші
	if listening then
		if input.UserInputType == Enum.UserInputType.Keyboard then
			local name = input.KeyCode == Enum.KeyCode.Backspace and "None" or input.KeyCode.Name
			local k = listening
			listening = nil
			setValue(k, name, true)
		end
		return
	end

	if input.UserInputType == Enum.UserInputType.MouseButton2 then
		aimHeld = true
	end

	if processed or not gateOpen then return end
	if input.UserInputType == Enum.UserInputType.Keyboard then
		for key, action in ACTIONS do
			if S[key] ~= "None" and input.KeyCode.Name == S[key] then
				action()
			end
		end
	end
end)

UserInputService.InputChanged:Connect(function(input)
	-- Слайдер
	if activeSlider then
		local same = input == activeSlider.input
		local mouseDrag = input.UserInputType == Enum.UserInputType.MouseMovement
			and activeSlider.input.UserInputType == Enum.UserInputType.MouseButton1
		if same or mouseDrag then
			activeSlider.update(input.Position.X)
		end
	end

	-- Перетягування вікна
	if dragging then
		local same = input == dragging.input
		local mouseDrag = input.UserInputType == Enum.UserInputType.MouseMovement
			and dragging.input.UserInputType == Enum.UserInputType.MouseButton1
		if same or mouseDrag then
			local d = Vector2.new(input.Position.X, input.Position.Y) - dragging.start
			UI.window.Position = UDim2.new(
				dragging.pos.X.Scale, dragging.pos.X.Offset + d.X,
				dragging.pos.Y.Scale, dragging.pos.Y.Offset + d.Y
			)
		end
	end
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton2 then
		aimHeld = false
	end
	if activeSlider and input == activeSlider.input then
		activeSlider.page.frame.ScrollingEnabled = true
		activeSlider = nil
	end
	if dragging and input == dragging.input then
		dragging = nil
	end
end)

local cam0 = getCamera()
if cam0 then
	cam0:GetPropertyChangedSignal("ViewportSize"):Connect(applyVisuals)
end

-- =====================================================================
-- БЛОК 15. ПРОТИ ДРОНІВ
-- Назви порівнюються "розмито": регістр, кирилиця/латиниця, цифри-підміни
-- (dr0ne), роздільники (D_r-o.n e) не мають значення. Суміш літер різних
-- алфавітів у назві теж розпізнається.
-- =====================================================================
do
	local CollectionService = game:GetService("CollectionService")
	local DRONE_TAG = "Drone"       -- модель із цим тегом завжди вважається дроном
	local MAX_DRONE_SIZE = 60       -- об'єкти більші за це (будівлі, карта) ігноруються
	local ARROWS = { "↑", "↗", "→", "↘", "↓", "↙", "←", "↖" }

	-- ---------- Нормалізація назв ----------
	local TRANSLIT = {
		[0x430] = "a", [0x431] = "b", [0x432] = "v", [0x433] = "g", [0x491] = "g",
		[0x434] = "d", [0x435] = "e", [0x454] = "e", [0x451] = "e", [0x436] = "zh",
		[0x437] = "z", [0x438] = "i", [0x456] = "i", [0x457] = "i", [0x439] = "y",
		[0x43A] = "k", [0x43B] = "l", [0x43C] = "m", [0x43D] = "n", [0x43E] = "o",
		[0x43F] = "p", [0x440] = "r", [0x441] = "s", [0x442] = "t", [0x443] = "u",
		[0x444] = "f", [0x445] = "kh", [0x446] = "ts", [0x447] = "ch", [0x448] = "sh",
		[0x449] = "shch", [0x44A] = "", [0x44B] = "y", [0x44C] = "", [0x44D] = "e",
		[0x44E] = "yu", [0x44F] = "ya",
	}
	local LEET = { ["0"] = "o", ["1"] = "i", ["3"] = "e", ["4"] = "a", ["5"] = "s", ["7"] = "t", ["@"] = "a", ["$"] = "s" }
	local foldCache = {}

	local function fold(s)
		local cached = foldCache[s]
		if cached then return cached end

		local out = {}
		local ok = pcall(function()
			for _, cp in utf8.codes(s) do
				local c = cp
				if c >= 0x410 and c <= 0x42F then c += 32
				elseif c >= 0x400 and c <= 0x40F then c += 80
				elseif c == 0x490 then c = 0x491 end

				if c < 128 then
					local ch = string.lower(string.char(c))
					ch = LEET[ch] or ch
					if string.find(ch, "^%w+$") then out[#out + 1] = ch end
				elseif TRANSLIT[c] then
					out[#out + 1] = TRANSLIT[c]
				end
			end
		end)

		local result
		if ok then
			result = table.concat(out)
		else
			result = string.gsub(string.lower(s), "[^%w]", "")
		end
		foldCache[s] = result
		return result
	end

	local function parseList(text)
		local list = {}
		for token in string.gmatch(tostring(text), "[^,;\n]+") do
			local f = fold(token)
			if #f >= 3 then list[#list + 1] = f end
		end
		return list
	end

	local function matches(name, include, exclude)
		local f = fold(name)
		for _, ex in exclude do
			if string.find(f, ex, 1, true) then return false end
		end
		for _, inc in include do
			if string.find(f, inc, 1, true) then return true end
		end
		return false
	end

	-- ---------- Пошук дронів на карті ----------
	local droneCache, enemyList = {}, {}
	local scanning, lastScan, forceScan = false, 0, true
	local lastEnemy = 0
	local lastNameKey = ""

	local function inAnyCharacter(inst)
		for _, plr in Players:GetPlayers() do
			local c = plr.Character
			if c and inst:IsDescendantOf(c) then return true end
		end
		return false
	end

	local function rescan()
		if scanning then return end
		scanning = true
		lastScan = os.clock()
		forceScan = false

		task.spawn(function()
			local include, exclude = parseList(S.DrName), parseList(S.DrExclude)
			local found, set, sizeCache = {}, {}, {}

			local function sizeOf(inst)
				local s = sizeCache[inst]
				if s then return s end
				if inst:IsA("Model") then
					s = inst:GetExtentsSize().Magnitude
				else
					s = inst.Size.Magnitude
				end
				sizeCache[inst] = s
				return s
			end

			local function consider(inst, checkSize)
				if not inst or not inst.Parent or set[inst] then return end
				if inAnyCharacter(inst) then return end

				local target = inst
				if inst:IsA("BasePart") then
					target = inst:FindFirstAncestorOfClass("Model") or inst
				end
				if checkSize and sizeOf(target) > MAX_DRONE_SIZE then
					-- велика модель (напр. будівля): беремо тільки сам дрон-деталь, якщо вона мала
					if inst:IsA("BasePart") and target ~= inst and sizeOf(inst) <= MAX_DRONE_SIZE then
						target = inst
					else
						return
					end
				end
				if not set[target] then
					set[target] = true
					found[#found + 1] = target
				end
			end

			-- 1) за тегом (від вашого скрипта дронів)
			for _, inst in CollectionService:GetTagged(DRONE_TAG) do
				consider(inst, false)
			end

			-- 2) за назвою (по шматках, щоб не підвисала гра)
			if #include > 0 then
				local n = 0
				for _, inst in workspace:GetDescendants() do
					n += 1
					if n % 3000 == 0 then task.wait() end
					if inst.Parent and (inst:IsA("Model") or inst:IsA("BasePart")) then
						if matches(inst.Name, include, exclude) then
							consider(inst, true)
						end
					end
				end
			end

			-- прибираємо вкладені (деталі всередині вже знайденого дрона)
			local out = {}
			for _, m in found do
				local nested = false
				local p = m.Parent
				while p and p ~= workspace do
					if set[p] then nested = true break end
					p = p.Parent
				end
				if not nested then out[#out + 1] = m end
			end

			droneCache = out
			scanning = false
		end)
	end

	-- ---------- Допоміжні ----------
	local function dronePos(m)
		if m:IsA("Model") then return m:GetPivot().Position end
		return m.Position
	end

	local function dronePart(m)
		if m:IsA("BasePart") then return m end
		return m.PrimaryPart or m:FindFirstChildWhichIsA("BasePart", true)
	end

	-- Ворожі дрони: без своїх і без дронів тімейтів
	local function rebuildEnemies()
		local out = {}
		for _, m in droneCache do
			if m.Parent then
				local own, team = false, false
				local uid = m:GetAttribute("OwnerUserId")
				if uid == player.UserId then own = true end

				local ownerPlr = uid and Players:GetPlayerByUserId(uid)
				if not ownerPlr then
					for _, plr in Players:GetPlayers() do
						if string.find(m.Name, plr.Name, 1, true) then
							ownerPlr = plr
							break
						end
					end
				end
				if ownerPlr == player then own = true end
				if ownerPlr and isTeammate(ownerPlr) then team = true end

				if not own and not team then out[#out + 1] = m end
			end
		end
		enemyList = out
	end

	droneReport = function()
		forceScan = true
		rescan()
		task.spawn(function()
			while scanning do task.wait(0.05) end
			rebuildEnemies()
			toast(string.format("Дронів знайдено: %d (ворожих: %d)", #droneCache, #enemyList), ACCENT, true)
		end)
	end

	-- ---------- Підсвітка ----------
	local espFolder = Instance.new("Folder")
	espFolder.Name = "FunModeDroneESP"
	espFolder.Parent = playerGui
	local dEsp = {}

	local function clearEsp()
		for m, e in dEsp do
			e.hl:Destroy()
			e.bb:Destroy()
			dEsp[m] = nil
		end
	end

	local function updateDroneEsp(rootPos)
		if not S.DrEsp then clearEsp() return end
		local seen = {}
		for _, m in enemyList do
			seen[m] = true
			local e = dEsp[m]
			if not e then
				local part = dronePart(m)
				if part then
					local hl = Instance.new("Highlight")
					hl.Adornee = m
					hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
					hl.FillTransparency = 0.5
					hl.Parent = espFolder

					local bb = Instance.new("BillboardGui")
					bb.Adornee = part
					bb.AlwaysOnTop = true
					bb.Size = UDim2.fromOffset(140, 18)
					bb.StudsOffset = Vector3.new(0, 2.5, 0)
					bb.Parent = espFolder

					local t = Instance.new("TextLabel")
					t.Size = UDim2.fromScale(1, 1)
					t.BackgroundTransparency = 1
					t.Font = Enum.Font.GothamBold
					t.TextSize = 13
					t.TextStrokeTransparency = 0.4
					t.Parent = bb

					e = { hl = hl, bb = bb, text = t }
					dEsp[m] = e
				end
			end
			if e then
				local c = colorOf(S.DrEspColor)
				e.hl.FillColor, e.hl.OutlineColor = c, c
				e.text.TextColor3 = c
				e.text.Text = string.format("ДРОН  %d ст", math.floor((dronePos(m) - rootPos).Magnitude))
			end
		end
		for m, e in dEsp do
			if not seen[m] then
				e.hl:Destroy()
				e.bb:Destroy()
				dEsp[m] = nil
			end
		end
	end

	-- ---------- Попередження: напис і червоне свічення по краях ----------
	local warnLabel = makeText(UI.gui, UDim2.fromOffset(320, 28), UDim2.new(0.5, 0, 0.16, 0),
		"", 18, Color3.fromRGB(255, 70, 70), Enum.Font.GothamBold, Enum.TextXAlignment.Center)
	warnLabel.AnchorPoint = Vector2.new(0.5, 0)
	warnLabel.TextStrokeTransparency = 0.4
	warnLabel.Visible = false

	local vig = Instance.new("Frame")
	vig.Size = UDim2.fromScale(1, 1)
	vig.BackgroundTransparency = 1
	vig.Active = false
	vig.Parent = UI.overlay
	local vigStroke = addStroke(vig, Color3.fromRGB(255, 40, 40), 14, 1)

	-- ---------- Головний цикл (після камери і основного прицілу) ----------
	local espTimer2, lastEvade, lastFire = 0, 0, 0

	RunService:BindToRenderStep("FunModeDrones", Enum.RenderPriority.Camera.Value + 2, function(dt)
		local anyOn = S.DrEsp or S.DrWarn or S.DrAim or S.DrAutoFire or S.DrEvade
		local cam = getCamera()
		local char = player.Character
		local root = getRoot(char)

		if not (anyOn and gateOpen and cam and root) then
			warnLabel.Visible = false
			vigStroke.Transparency = 1
			clearEsp()
			return
		end

		-- зміна списку назв у меню → одразу пересканувати
		local nameKey = S.DrName .. "|" .. S.DrExclude
		if nameKey ~= lastNameKey then
			lastNameKey = nameKey
			forceScan = true
		end
		if (forceScan or os.clock() - lastScan > 4) and not scanning then
			rescan()
		end
		if os.clock() - lastEnemy > 0.25 then
			lastEnemy = os.clock()
			rebuildEnemies()
		end

		local camPos = cam.CFrame.Position

		-- найближчий до гравця дрон
		local nearest, nearestDist
		for _, m in enemyList do
			if m.Parent then
				local d = (dronePos(m) - root.Position).Magnitude
				if not nearestDist or d < nearestDist then nearest, nearestDist = m, d end
			end
		end

		-- підсвітка (10 разів на секунду)
		espTimer2 += dt
		if espTimer2 >= 0.1 then
			espTimer2 = 0
			updateDroneEsp(root.Position)
		end

		-- попередження
		if S.DrWarn and nearest and nearestDist <= S.DrWarnRadius then
			local rel = cam.CFrame:PointToObjectSpace(dronePos(nearest))
			local a = math.atan2(rel.X, -rel.Z)
			local idx = math.floor((a + math.pi / 8) / (math.pi / 4)) % 8 + 1
			warnLabel.Text = string.format("⚠ ДРОН  %s  %d ст", ARROWS[idx], math.floor(nearestDist))
			warnLabel.Visible = true
			vigStroke.Transparency = 0.55 + 0.35 * math.sin(os.clock() * 10)
		else
			warnLabel.Visible = false
			vigStroke.Transparency = 1
		end

		-- автоприціл і автовогонь по дронах
		local aimActive = S.DrAim and (not S.AimHold or not hasMouse or aimHeld)
		if aimActive or S.DrAutoFire then
			local point = getAimPoint()
			local bestPos, bestScore
			for _, m in enemyList do
				if m.Parent then
					local pos = dronePos(m)
					if (pos - camPos).Magnitude <= S.DrAimRange then
						local sp, onScreen = cam:WorldToViewportPoint(pos)
						if onScreen then
							local d2 = (Vector2.new(sp.X, sp.Y) - point).Magnitude
							if not bestScore or d2 < bestScore then
								bestPos, bestScore = pos, d2
							end
						end
					end
				end
			end

			if bestPos then
				if aimActive then
					cam.CFrame = cam.CFrame:Lerp(CFrame.lookAt(camPos, bestPos), math.max(S.Smooth, 0.35))
				end
				if S.DrAutoFire and os.clock() - lastFire > 0.12 then
					local dir = (bestPos - camPos).Unit
					if dir:Dot(cam.CFrame.LookVector) > 0.9962 then -- у межах ~5° від прицілу
						local tool = char:FindFirstChildOfClass("Tool")
						if tool then
							lastFire = os.clock()
							tool:Activate()
						end
					end
				end
			end
		end

		-- авто-ухилення від дрона
		if S.DrEvade and nearest and nearestDist <= S.DrEvadeRadius
			and os.clock() - lastEvade >= S.DrEvadeCd and isAlive(char) then
			lastEvade = os.clock()
			local dpos = dronePos(nearest)
			local away = Vector3.new(root.Position.X - dpos.X, 0, root.Position.Z - dpos.Z)
			away = away.Magnitude > 0.1 and away.Unit or Vector3.new(1, 0, 0)

			local params = RaycastParams.new()
			params.FilterType = Enum.RaycastFilterType.Exclude
			params.FilterDescendantsInstances = { char }

			for _ = 1, 8 do
				local dir = CFrame.Angles(0, math.rad(math.random(-60, 60)), 0) * away
				local origin = root.Position + dir * S.DrEvadeDist + Vector3.new(0, 40, 0)
				local hit = workspace:Raycast(origin, Vector3.new(0, -150, 0), params)
				if hit and hit.Normal.Y > 0.7 then
					char:PivotTo(CFrame.new(hit.Position + Vector3.new(0, 4, 0)))
					root.AssemblyLinearVelocity = Vector3.zero
					break
				end
			end
		end
	end)
end
