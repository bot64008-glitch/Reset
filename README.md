-- Auto Reset Loop | чёрный / жёлтый / серый / белый
-- Цикл: ждём респавн -> пауза -> ресет (клик PlayButton) -> ждём респавн -> ...

if _G.ARL_Destroy then pcall(_G.ARL_Destroy) end

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local VIM = game:GetService("VirtualInputManager")
local player = Players.LocalPlayer

----------------------------------------------------------------
-- НАСТРОЙКИ (меняются и в меню, раздел "Настройки")
----------------------------------------------------------------
local CFG = {
	Delay = 1.5,          -- пауза после респавна перед ресетом (сек)
	Timeout = 15,         -- сколько ждать респавн, прежде чем повторить ресет (сек)
	Method = "Play",      -- "Play" = клик PlayButton, "Health" = Humanoid.Health = 0
	Running = false,
}

local alive = true
local conns = {}
local count = 0

----------------------------------------------------------------
-- ЦВЕТА
----------------------------------------------------------------
local C = {
	black = Color3.fromRGB(14, 14, 14),
	dark = Color3.fromRGB(32, 32, 32),
	gray = Color3.fromRGB(70, 70, 70),
	lgray = Color3.fromRGB(150, 150, 150),
	yellow = Color3.fromRGB(255, 200, 0),
	white = Color3.fromRGB(245, 245, 245),
}

----------------------------------------------------------------
-- GUI helpers
----------------------------------------------------------------
local function mk(class, props, parent)
	local o = Instance.new(class)
	for k, v in pairs(props) do o[k] = v end
	o.Parent = parent
	return o
end
local function round(o, r) mk("UICorner", { CornerRadius = UDim.new(0, r or 6) }, o) end
local function stroke(o, c, t)
	mk("UIStroke", { Color = c, Thickness = t or 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, o)
end
local function label(parent, text, pos, size, color, align, ts)
	return mk("TextLabel", {
		Text = text, Position = pos, Size = size, BackgroundTransparency = 1,
		TextColor3 = color or C.white, Font = Enum.Font.GothamBold, TextSize = ts or 12,
		TextXAlignment = align or Enum.TextXAlignment.Left,
	}, parent)
end
local function button(parent, text, pos, size, bg, fg, ts)
	local b = mk("TextButton", {
		Text = text, Position = pos, Size = size, BackgroundColor3 = bg or C.gray,
		TextColor3 = fg or C.white, Font = Enum.Font.GothamBold, TextSize = ts or 12,
		AutoButtonColor = true, BorderSizePixel = 0,
	}, parent)
	round(b, 5)
	return b
end

local function parentGui()
	local ok, h = pcall(function() return gethui and gethui() end)
	if ok and h then return h end
	local ok2, cg = pcall(function() return game:GetService("CoreGui") end)
	if ok2 and cg then return cg end
	return player:WaitForChild("PlayerGui")
end

----------------------------------------------------------------
-- ОКНО
----------------------------------------------------------------
local gui = mk("ScreenGui", {
	Name = "AutoResetLoop", ResetOnSpawn = false, DisplayOrder = 999,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling, IgnoreGuiInset = true,
}, nil)
gui.Parent = parentGui()

local W, H = 190, 142
local root = mk("Frame", {
	Size = UDim2.fromOffset(W, H), Position = UDim2.new(0, 12, 0.3, 0),
	BackgroundColor3 = C.black, BorderSizePixel = 0,
}, gui)
round(root, 8)
stroke(root, C.yellow, 1.5)

-- шапка
local header = mk("Frame", { Size = UDim2.new(1, 0, 0, 26), BackgroundColor3 = C.yellow, BorderSizePixel = 0 }, root)
round(header, 8)
mk("Frame", { Size = UDim2.new(1, 0, 0, 8), Position = UDim2.new(0, 0, 1, -8), BackgroundColor3 = C.yellow, BorderSizePixel = 0 }, header)
label(header, "  AUTO RESET", UDim2.new(0, 0, 0, 0), UDim2.new(1, -60, 1, 0), C.black, nil, 12)
local minBtn = button(header, "—", UDim2.new(1, -54, 0, 3), UDim2.fromOffset(24, 20), C.black, C.yellow, 14)
local closeBtn = button(header, "✕", UDim2.new(1, -28, 0, 3), UDim2.fromOffset(24, 20), C.black, C.white, 12)

-- кружок в свёрнутом виде
local bubble = button(root, "AR", UDim2.new(0, 0, 0, 0), UDim2.new(1, 0, 1, 0), C.yellow, C.black, 13)
bubble.Visible = false
bubble.AutoButtonColor = false

-- вкладки
local body = mk("Frame", { Size = UDim2.new(1, 0, 1, -26), Position = UDim2.new(0, 0, 0, 26), BackgroundTransparency = 1 }, root)
local tabMain = button(body, "Главная", UDim2.fromOffset(6, 4), UDim2.fromOffset(86, 20), C.yellow, C.black, 11)
local tabSet = button(body, "Настройки", UDim2.fromOffset(98, 4), UDim2.fromOffset(86, 20), C.dark, C.white, 11)

local pageMain = mk("Frame", { Size = UDim2.new(1, -12, 1, -32), Position = UDim2.fromOffset(6, 28), BackgroundTransparency = 1 }, body)
local pageSet = mk("Frame", { Size = UDim2.new(1, -12, 1, -32), Position = UDim2.fromOffset(6, 28), BackgroundTransparency = 1, Visible = false }, body)

local function showTab(main)
	pageMain.Visible = main
	pageSet.Visible = not main
	tabMain.BackgroundColor3 = main and C.yellow or C.dark
	tabMain.TextColor3 = main and C.black or C.white
	tabSet.BackgroundColor3 = main and C.dark or C.yellow
	tabSet.TextColor3 = main and C.white or C.black
end
tabMain.Activated:Connect(function() showTab(true) end)
tabSet.Activated:Connect(function() showTab(false) end)

----------------------------------------------------------------
-- ГЛАВНАЯ
----------------------------------------------------------------
local statusLbl = label(pageMain, "Статус: ожидание", UDim2.new(0, 0, 0, 0), UDim2.new(1, 0, 0, 18), C.white, nil, 12)
local countLbl = label(pageMain, "Ресетов: 0", UDim2.new(0, 0, 0, 18), UDim2.new(1, 0, 0, 16), C.lgray, nil, 11)
local toggleBtn = button(pageMain, "▶  СТАРТ", UDim2.new(0, 0, 0, 40), UDim2.new(1, 0, 0, 36), C.yellow, C.black, 15)

local function setStatus(t) statusLbl.Text = "Статус: " .. t end

local function refreshToggle()
	if CFG.Running then
		toggleBtn.Text = "■  СТОП"
		toggleBtn.BackgroundColor3 = C.gray
		toggleBtn.TextColor3 = C.yellow
	else
		toggleBtn.Text = "▶  СТАРТ"
		toggleBtn.BackgroundColor3 = C.yellow
		toggleBtn.TextColor3 = C.black
		setStatus("пауза")
	end
end
toggleBtn.Activated:Connect(function()
	CFG.Running = not CFG.Running
	refreshToggle()
end)

----------------------------------------------------------------
-- НАСТРОЙКИ
----------------------------------------------------------------
local function stepper(y, text, key, step, minv, maxv)
	label(pageSet, text, UDim2.fromOffset(0, y), UDim2.fromOffset(80, 22), C.white, nil, 11)
	local minus = button(pageSet, "−", UDim2.fromOffset(82, y), UDim2.fromOffset(24, 22), C.gray, C.white, 14)
	local box = mk("TextBox", {
		Text = tostring(CFG[key]), Position = UDim2.fromOffset(108, y), Size = UDim2.fromOffset(44, 22),
		BackgroundColor3 = C.dark, TextColor3 = C.yellow, Font = Enum.Font.GothamBold, TextSize = 12,
		ClearTextOnFocus = false, BorderSizePixel = 0,
	}, pageSet)
	round(box, 5)
	local plus = button(pageSet, "+", UDim2.fromOffset(154, y), UDim2.fromOffset(24, 22), C.gray, C.white, 14)
	local function set(v)
		v = math.clamp(math.floor(v * 10 + 0.5) / 10, minv, maxv)
		CFG[key] = v
		box.Text = tostring(v)
	end
	minus.Activated:Connect(function() set(CFG[key] - step) end)
	plus.Activated:Connect(function() set(CFG[key] + step) end)
	box.FocusLost:Connect(function() set(tonumber(box.Text) or CFG[key]) end)
end
stepper(0, "Пауза, сек", "Delay", 0.1, 0, 60)
stepper(26, "Таймаут, сек", "Timeout", 1, 3, 120)

local methodBtn = button(pageSet, "Метод: кнопка Play", UDim2.fromOffset(0, 54), UDim2.new(1, 0, 0, 24), C.dark, C.white, 11)
stroke(methodBtn, C.yellow, 1)
methodBtn.Activated:Connect(function()
	if CFG.Method == "Play" then
		CFG.Method = "Health"
		methodBtn.Text = "Метод: Health = 0"
	else
		CFG.Method = "Play"
		methodBtn.Text = "Метод: кнопка Play"
	end
end)

----------------------------------------------------------------
-- СВОРАЧИВАНИЕ / ЗАКРЫТИЕ / ПЕРЕТАСКИВАНИЕ
----------------------------------------------------------------
local minimized = false
local function setMin(v)
	minimized = v
	header.Visible = not v
	body.Visible = not v
	bubble.Visible = v
	root.Size = v and UDim2.fromOffset(40, 40) or UDim2.fromOffset(W, H)
end
minBtn.Activated:Connect(function() setMin(true) end)

local function destroy()
	alive = false
	CFG.Running = false
	for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
	pcall(function() gui:Destroy() end)
	_G.ARL_Destroy = nil
end
_G.ARL_Destroy = destroy
closeBtn.Activated:Connect(destroy)

local function draggable(handle, onTap)
	local dragging, moved = false, false
	local startPos, startFrame
	table.insert(conns, handle.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then
			dragging, moved = true, false
			startPos, startFrame = i.Position, root.Position
			local ch
			ch = i.Changed:Connect(function()
				if i.UserInputState == Enum.UserInputState.End then
					dragging = false
					ch:Disconnect()
					if not moved and onTap then onTap() end
				end
			end)
		end
	end))
	table.insert(conns, UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseMovement) then
			local d = i.Position - startPos
			if d.Magnitude > 6 then moved = true end
			if moved then
				root.Position = UDim2.new(startFrame.X.Scale, startFrame.X.Offset + d.X, startFrame.Y.Scale, startFrame.Y.Offset + d.Y)
			end
		end
	end))
end
draggable(header)
draggable(bubble, function() setMin(false) end)

----------------------------------------------------------------
-- ЛОГИКА
----------------------------------------------------------------
local function pressPlay()
	local ok = pcall(function()
		local btn = player.PlayerGui:WaitForChild("SpawnGui", 3):WaitForChild("Customization", 3):WaitForChild("PlayButton", 3)
		local pos = btn.AbsolutePosition + btn.AbsoluteSize / 2
		VIM:SendMouseButtonEvent(pos.X, pos.Y, 0, true, game, 0)
		VIM:SendMouseButtonEvent(pos.X, pos.Y, 0, false, game, 0)
	end)
	return ok
end

local function doReset()
	if CFG.Method == "Health" then
		local ch = player.Character
		local hum = ch and ch:FindFirstChildOfClass("Humanoid")
		if hum then hum.Health = 0 end
	else
		pressPlay()
	end
end

local function isReady(ch)
	if not ch then return false end
	local hum = ch:FindFirstChildOfClass("Humanoid")
	return hum and hum.Health > 0 and ch:FindFirstChild("HumanoidRootPart") ~= nil
end

-- ждём нового (или готового) персонажа; oldChar = персонаж, которого НЕ считаем
local function waitRespawn(oldChar)
	local t0 = os.clock()
	while alive and CFG.Running do
		local ch = player.Character
		if ch ~= oldChar and isReady(ch) then return true end
		if os.clock() - t0 > CFG.Timeout then return false end
		task.wait(0.1)
	end
	return false
end

local function sleep(sec)
	local t0 = os.clock()
	while alive and CFG.Running and os.clock() - t0 < sec do
		task.wait(0.05)
	end
end

task.spawn(function()
	while alive do
		if CFG.Running then
			-- 1) убеждаемся, что персонаж есть
			if not isReady(player.Character) then
				setStatus("ждём респавн")
				waitRespawn(nil)
			end
			-- 2) пауза
			if CFG.Running and alive then
				setStatus("пауза " .. CFG.Delay .. "с")
				sleep(CFG.Delay)
			end
			-- 3) ресет
			if CFG.Running and alive then
				local old = player.Character
				setStatus("ресет")
				doReset()
				count += 1
				countLbl.Text = "Ресетов: " .. count
				-- 4) ждём респавн
				setStatus("ждём респавн")
				if not waitRespawn(old) and CFG.Running then
					setStatus("таймаут, повтор")
				end
			end
		else
			task.wait(0.2)
		end
	end
end)

refreshToggle()
