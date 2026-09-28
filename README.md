-- FLUX PING LAGGER 180x240 | Compact + Sliding Auto Activate
local Players          = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local HttpService      = game:GetService("HttpService")
local RunService       = game:GetService("RunService")
local TweenService     = game:GetService("TweenService")

local plr = Players.LocalPlayer
local playerGui = plr:WaitForChild("PlayerGui")

-------------------------------------------------
-- CONFIG & SAVE
-------------------------------------------------
local CONFIG_FILE = "FluxPingLagger_Config.json"

local DEFAULT_CFG = {
	power        = 100000,
	interval     = 0.125,
	keybind      = "F",
	autoBrainrot = true,
}

local cfg = {
	power        = DEFAULT_CFG.power,
	interval     = DEFAULT_CFG.interval,
	keybind      = DEFAULT_CFG.keybind,
	autoBrainrot = DEFAULT_CFG.autoBrainrot,
}

local function resolveKey(name)
	if not name or name == "" or name == "None" then return nil end
	local ok, val = pcall(function() return Enum.KeyCode[name] end)
	return (ok and val) or nil
end

local function saveConfig()
	local ok, encoded = pcall(function() return HttpService:JSONEncode(cfg) end)
	if ok and encoded and writefile then
		pcall(writefile, CONFIG_FILE, encoded)
	end
end

local function loadConfig()
	if not (isfile and readfile and isfile(CONFIG_FILE)) then return end
	local ok, data = pcall(function() return HttpService:JSONDecode(readfile(CONFIG_FILE)) end)
	if not ok or type(data) ~= "table" then return end

	cfg.power        = tonumber(data.power) or DEFAULT_CFG.power
	cfg.interval     = tonumber(data.interval) or DEFAULT_CFG.interval
	cfg.keybind      = type(data.keybind) == "string" and data.keybind or DEFAULT_CFG.keybind
	cfg.autoBrainrot = type(data.autoBrainrot) == "boolean" and data.autoBrainrot or DEFAULT_CFG.autoBrainrot
end

loadConfig()

-------------------------------------------------
-- STATE
-------------------------------------------------
local active            = false
local listeningFor      = false
local remote            = nil
local brainrotMode      = false
local lastBrainrotState = false
local manualOverride    = false

local BLACKLISTED = {
	[Enum.KeyCode.Escape]      = true,
	[Enum.KeyCode.LeftControl] = true,
	[Enum.KeyCode.Unknown]     = true,
}

-------------------------------------------------
-- PING LAGGER CORE
-------------------------------------------------
local function findRemote()
	local rrs = game:FindFirstChild("RobloxReplicatedStorage")
	if not rrs then return nil end

	for _, name in ipairs({"SetPlayerBlockList", "UpdatePlayerBlockList", "SetBlockList", "UpdateBlockList"}) do
		local r = rrs:FindFirstChild(name)
		if r and r:IsA("RemoteEvent") then
			return r
		end
	end

	for _, c in ipairs(rrs:GetChildren()) do
		if c:IsA("RemoteEvent") and c.Name:find("Block") then
			return c
		end
	end
	return nil
end

remote = findRemote()

local function buildPayload(power)
	local main = {}
	local nested = {{}}
	local current = nested[1]
	for _ = 1, 186 do
		local n = {}
		table.insert(current, n)
		current = n
	end
	local maxRep = math.min(math.floor(power / 188), 10000)
	for _ = 1, maxRep do
		table.insert(main, nested)
	end
	return main
end

local function runPingLoop()
	local delay = cfg.interval
	while active and remote do
		local payload = buildPayload(cfg.power)
		local ok = pcall(function() remote:FireServer(payload) end)
		if not ok then
			delay = math.min(delay * 1.5, 0.5)
		else
			delay = math.max(delay * 0.995, 0.05)
		end
		task.wait(delay)
	end
end

local function setLag(state)
	active = state
	if active then
		if not remote then
			remote = findRemote()
			if not remote then
				active = false
				return
			end
		end
		task.spawn(runPingLoop)
	end
end

-------------------------------------------------
-- AUTO ACTIVATE DETECTION
-------------------------------------------------
RunService.Heartbeat:Connect(function()
	if not cfg.autoBrainrot then
		if brainrotMode then
			brainrotMode = false
			lastBrainrotState = false
		end
		return
	end

	local char = plr.Character
	if not char then return end
	local hum = char:FindFirstChild("Humanoid")
	if not hum then return end

	local hasBrainrot = hum.WalkSpeed < 25

	if hasBrainrot and not lastBrainrotState then
		brainrotMode = true
		lastBrainrotState = true
		manualOverride = false
		setLag(true)
	elseif not hasBrainrot and lastBrainrotState then
		brainrotMode = false
		lastBrainrotState = false
		manualOverride = false
		setLag(false)
	end
end)

-------------------------------------------------
-- GUI
-------------------------------------------------
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FluxPingLagger"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Name = "Main"
frame.Size = UDim2.new(0, 180, 0, 240)
frame.Position = UDim2.new(0.5, -90, 0.5, -120)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
frame.BorderSizePixel = 0
frame.Visible = true
frame.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 14)
corner.Parent = frame

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(255, 255, 255)
stroke.Transparency = 0.85
stroke.Thickness = 1
stroke.Parent = frame

local bgImage = Instance.new("ImageLabel")
bgImage.Size = UDim2.new(1, 0, 1, 0)
bgImage.BackgroundTransparency = 1
bgImage.Image = "rbxassetid://86667711139501"
bgImage.ScaleType = Enum.ScaleType.Stretch
bgImage.Parent = frame

local imageCorner = Instance.new("UICorner")
imageCorner.CornerRadius = UDim.new(0, 14)
imageCorner.Parent = bgImage

local overlay = Instance.new("Frame")
overlay.Size = UDim2.new(1, 0, 1, 0)
overlay.BackgroundColor3 = Color3.fromRGB(10, 10, 18)
overlay.BackgroundTransparency = 0.38
overlay.BorderSizePixel = 0
overlay.ZIndex = 2
overlay.Parent = frame

local overlayCorner = Instance.new("UICorner")
overlayCorner.CornerRadius = UDim.new(0, 14)
overlayCorner.Parent = overlay

-- Title bar
local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1, 0, 0, 26)
titleBar.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
titleBar.BackgroundTransparency = 0.4
titleBar.BorderSizePixel = 0
titleBar.ZIndex = 5
titleBar.Parent = frame

local titleCorner = Instance.new("UICorner")
titleCorner.CornerRadius = UDim.new(0, 14)
titleCorner.Parent = titleBar

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -70, 1, 0)
title.Position = UDim2.new(0, 8, 0, 0)
title.BackgroundTransparency = 1
title.Text = "FLUX PING LAGGER"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextSize = 11
title.Font = Enum.Font.GothamBold
title.TextXAlignment = Enum.TextXAlignment.Left
title.ZIndex = 6
title.Parent = titleBar

-- Tiny Keybind button
local keybindBtn = Instance.new("TextButton")
keybindBtn.Size = UDim2.new(0, 32, 0, 16)
keybindBtn.Position = UDim2.new(1, -58, 0.5, -8)
keybindBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
keybindBtn.Text = cfg.keybind
keybindBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
keybindBtn.TextSize = 9
keybindBtn.Font = Enum.Font.GothamBold
keybindBtn.ZIndex = 7
keybindBtn.Parent = titleBar

local keybindCorner = Instance.new("UICorner")
keybindCorner.CornerRadius = UDim.new(0, 4)
keybindCorner.Parent = keybindBtn

-- Close button
local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 18, 0, 18)
closeBtn.Position = UDim2.new(1, -22, 0.5, -9)
closeBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
closeBtn.Text = "-"
closeBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
closeBtn.TextSize = 14
closeBtn.Font = Enum.Font.GothamBold
closeBtn.ZIndex = 7
closeBtn.Parent = titleBar

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(1, 0)
closeCorner.Parent = closeBtn

-------------------------------------------------
-- COMPACT OPTIONS
-------------------------------------------------
local function createLabel(text, y)
	local lbl = Instance.new("TextLabel")
	lbl.Size = UDim2.new(0, 70, 0, 12)
	lbl.Position = UDim2.new(0, 10, 0, y)
	lbl.BackgroundTransparency = 1
	lbl.Text = text
	lbl.TextColor3 = Color3.fromRGB(220, 220, 230)
	lbl.TextSize = 10
	lbl.Font = Enum.Font.Gotham
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.ZIndex = 5
	lbl.Parent = frame
	return lbl
end

local function createTextBox(text, y)
	local box = Instance.new("TextBox")
	box.Size = UDim2.new(0, 90, 0, 18)
	box.Position = UDim2.new(0, 80, 0, y)
	box.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
	box.Text = text
	box.TextColor3 = Color3.fromRGB(0, 0, 0)
	box.PlaceholderColor3 = Color3.fromRGB(100, 100, 120)
	box.TextSize = 11
	box.Font = Enum.Font.GothamBold
	box.ClearTextOnFocus = false
	box.ZIndex = 5
	box.Parent = frame

	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, 5)
	c.Parent = box
	return box
end

-- Power
createLabel("power:", 36)
local powerBox = createTextBox(tostring(cfg.power), 34)
powerBox.PlaceholderText = "100000"

-- Delay
createLabel("delay:", 62)
local intervalBox = createTextBox(string.format("%.3f", cfg.interval), 60)
intervalBox.PlaceholderText = "0.125"

-- Auto Activate label + Smaller Sliding Toggle
createLabel("Auto Activate", 92)

local toggleTrack = Instance.new("Frame")
toggleTrack.Size = UDim2.new(0, 34, 0, 16)          -- smaller
toggleTrack.Position = UDim2.new(0, 136, 0, 90)
toggleTrack.BackgroundColor3 = cfg.autoBrainrot and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(40, 40, 50)
toggleTrack.BorderSizePixel = 0
toggleTrack.ZIndex = 5
toggleTrack.Parent = frame

local trackCorner = Instance.new("UICorner")
trackCorner.CornerRadius = UDim.new(1, 0)
trackCorner.Parent = toggleTrack

local toggleKnob = Instance.new("Frame")
toggleKnob.Size = UDim2.new(0, 12, 0, 12)           -- smaller
toggleKnob.Position = cfg.autoBrainrot and UDim2.new(1, -14, 0.5, -6) or UDim2.new(0, 2, 0.5, -6)
toggleKnob.BackgroundColor3 = cfg.autoBrainrot and Color3.fromRGB(0, 0, 0) or Color3.fromRGB(200, 200, 210)
toggleKnob.BorderSizePixel = 0
toggleKnob.ZIndex = 6
toggleKnob.Parent = toggleTrack

local knobCorner = Instance.new("UICorner")
knobCorner.CornerRadius = UDim.new(1, 0)
knobCorner.Parent = toggleKnob

local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(1, 0, 1, 0)
toggleBtn.BackgroundTransparency = 1
toggleBtn.Text = ""
toggleBtn.ZIndex = 7
toggleBtn.Parent = toggleTrack

-- Larger Activate / Deactivate button
local lagBtn = Instance.new("TextButton")
lagBtn.Size = UDim2.new(0, 160, 0, 36)
lagBtn.Position = UDim2.new(0, 10, 0, 125)
lagBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
lagBtn.Text = active and "DEACTIVATE" or "ACTIVATE"
lagBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
lagBtn.TextSize = 14
lagBtn.Font = Enum.Font.GothamBold
lagBtn.ZIndex = 5
lagBtn.Parent = frame

local lagCorner = Instance.new("UICorner")
lagCorner.CornerRadius = UDim.new(0, 8)
lagCorner.Parent = lagBtn

-------------------------------------------------
-- TOGGLE BUTTON (always starts on the left)
-------------------------------------------------
local toggle = Instance.new("TextButton")
toggle.Name = "Toggle"
toggle.Size = UDim2.new(0, 64, 0, 26)
toggle.Position = UDim2.new(0, 12, 0.5, -13)   -- left side of screen
toggle.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
toggle.Text = ""
toggle.Visible = false
toggle.ZIndex = 20
toggle.Parent = screenGui

local toggleCorner2 = Instance.new("UICorner")
toggleCorner2.CornerRadius = UDim.new(0, 8)
toggleCorner2.Parent = toggle

local toggleStroke = Instance.new("UIStroke")
toggleStroke.Color = Color3.fromRGB(255, 255, 255)
toggleStroke.Transparency = 0.75
toggleStroke.Thickness = 1
toggleStroke.Parent = toggle

local toggleImage = Instance.new("ImageLabel")
toggleImage.Size = UDim2.new(1, 0, 1, 0)
toggleImage.BackgroundTransparency = 1
toggleImage.Image = "rbxassetid://86667711139501"
toggleImage.ScaleType = Enum.ScaleType.Stretch
toggleImage.ZIndex = 21
toggleImage.Parent = toggle

local toggleImageCorner = Instance.new("UICorner")
toggleImageCorner.CornerRadius = UDim.new(0, 8)
toggleImageCorner.Parent = toggleImage

-------------------------------------------------
-- SMOOTH DRAGGING
-------------------------------------------------
local function makeDraggable(guiObject, dragHandle)
	local dragging, dragStart, startPos = false, nil, nil

	dragHandle.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragStart = input.Position
			startPos = guiObject.Position
			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
				end
			end)
		end
	end)

	UserInputService.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			local delta = input.Position - dragStart
			guiObject.Position = UDim2.new(
				startPos.X.Scale, startPos.X.Offset + delta.X,
				startPos.Y.Scale, startPos.Y.Offset + delta.Y
			)
		end
	end)
end

makeDraggable(frame, titleBar)
makeDraggable(toggle, toggle)

-------------------------------------------------
-- UI + LOGIC
-------------------------------------------------
local function updateToggleVisual()
	local goalPos = cfg.autoBrainrot and UDim2.new(1, -14, 0.5, -6) or UDim2.new(0, 2, 0.5, -6)
	local goalColor = cfg.autoBrainrot and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(40, 40, 50)
	local knobColor = cfg.autoBrainrot and Color3.fromRGB(0, 0, 0) or Color3.fromRGB(200, 200, 210)

	TweenService:Create(toggleKnob, TweenInfo.new(0.18, Enum.EasingStyle.Quad), {Position = goalPos}):Play()
	TweenService:Create(toggleTrack, TweenInfo.new(0.18, Enum.EasingStyle.Quad), {BackgroundColor3 = goalColor}):Play()
	TweenService:Create(toggleKnob, TweenInfo.new(0.18, Enum.EasingStyle.Quad), {BackgroundColor3 = knobColor}):Play()
end

local function refreshUI()
	powerBox.Text = tostring(cfg.power)
	intervalBox.Text = string.format("%.3f", cfg.interval)
	keybindBtn.Text = listeningFor and "..." or cfg.keybind

	updateToggleVisual()

	lagBtn.Text = active and "DEACTIVATE" or "ACTIVATE"
	if active then
		lagBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
		lagBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
	else
		lagBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
		lagBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	end
end

powerBox.FocusLost:Connect(function()
	local num = tonumber(powerBox.Text)
	if num and num >= 1 then
		cfg.power = math.floor(num)
		saveConfig()
	end
	refreshUI()
end)

intervalBox.FocusLost:Connect(function()
	local num = tonumber(intervalBox.Text)
	if num and num >= 0.01 then
		cfg.interval = num
		saveConfig()
	end
	refreshUI()
end)

keybindBtn.MouseButton1Click:Connect(function()
	listeningFor = true
	refreshUI()
end)

toggleBtn.MouseButton1Click:Connect(function()
	cfg.autoBrainrot = not cfg.autoBrainrot
	if not cfg.autoBrainrot then
		brainrotMode = false
		lastBrainrotState = false
	end
	saveConfig()
	refreshUI()
end)

lagBtn.MouseButton1Click:Connect(function()
	local newState = not active
	if brainrotMode then
		manualOverride = not newState
	else
		manualOverride = false
	end
	setLag(newState)
	refreshUI()
end)

closeBtn.MouseButton1Click:Connect(function()
	frame.Visible = false
	toggle.Visible = true
end)

toggle.MouseButton1Click:Connect(function()
	toggle.Visible = false
	frame.Visible = true
end)

UserInputService.InputBegan:Connect(function(input, processed)
	local kc = input.KeyCode
	if kc == Enum.KeyCode.Unknown then return end

	if listeningFor then
		if kc == Enum.KeyCode.Escape then
			listeningFor = false
			refreshUI()
			return
		end
		if not BLACKLISTED[kc] then
			cfg.keybind = kc.Name
			listeningFor = false
			saveConfig()
			refreshUI()
		end
		return
	end

	if processed then return end

	local keyEnum = resolveKey(cfg.keybind)
	if keyEnum and kc == keyEnum then
		local newState = not active
		if brainrotMode then
			manualOverride = not newState
		else
			manualOverride = false
		end
		setLag(newState)
		refreshUI()
	end
end)

refreshUI()
