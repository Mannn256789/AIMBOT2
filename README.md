--// RAIDEN 

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TeamsService = game:GetService("Teams")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local SAVE_FILE = "raiden_config.json"

local function saveToLocalStorage(data)
	local ok, encoded = pcall(function()
		return HttpService:JSONEncode(data)
	end)
	if ok then
		writefile(SAVE_FILE, encoded)
	end
end

local function loadFromLocalStorage()
	if not isfile(SAVE_FILE) then return {} end
	local ok, data = pcall(function()
		return HttpService:JSONDecode(readfile(SAVE_FILE))
	end)
	if ok and type(data) == "table" then
		return data
	end
	return {}
end

local b64chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
local function base64Encode(data)
	local bytes = {}
	for i = 1, #data do
		bytes[#bytes + 1] = string.byte(data, i)
	end
	local result = {}
	for i = 1, #bytes, 3 do
		local b1, b2, b3 = bytes[i], bytes[i+1], bytes[i+2]
		local n = (b1 or 0) * 0x10000 + (b2 or 0) * 0x100 + (b3 or 0)
		local c1 = math.floor(n / 0x40000) % 0x40
		local c2 = math.floor(n / 0x1000) % 0x40
		local c3 = math.floor(n / 0x40) % 0x40
		local c4 = n % 0x40
		result[#result + 1] = b64chars:sub(c1 + 1, c1 + 1)
		result[#result + 1] = b64chars:sub(c2 + 1, c2 + 1)
		if b2 then
			result[#result + 1] = b64chars:sub(c3 + 1, c3 + 1)
		else
			result[#result + 1] = '='
		end
		if b3 then
			result[#result + 1] = b64chars:sub(c4 + 1, c4 + 1)
		else
			result[#result + 1] = '='
		end
	end
	return table.concat(result)
end

local function base64Decode(data)
	local result = {}
	local n = 0
	local bits = 0
	for i = 1, #data do
		local c = data:sub(i, i)
		if c == '=' then
			break
		end
		local idx = b64chars:find(c) - 1
		if idx then
			n = n * 64 + idx
			bits = bits + 6
			if bits >= 8 then
				bits = bits - 8
				local b = math.floor(n / (2 ^ bits)) % 256
				result[#result + 1] = string.char(b)
			end
		end
	end
	return table.concat(result)
end

local cfg = {
	lockSpeed = 0.55,
	fovRadius = 180,
	maxDistance = 300,
	prediction = 0.05,
	locked = false,
	teamCheck = false,
	wallCheck = true,
	keybind = Enum.KeyCode.Q,
}

local esp = {
	enabled = false,
	showHealth = true,
	showName = true,
	showDistance = true,
	fovEnabled = true, 
	targetTeams = {},
	targetPlayers = {},
}

local target = nil
local savedConfigs = {}
local cfgCount = 0
local espDrawings = {}

local playerGui = player:WaitForChild("PlayerGui")
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "RAIDEN_GUI"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = playerGui

local C = {
	bg = Color3.fromRGB(8, 8, 8),
	surface = Color3.fromRGB(18, 18, 18),
	accent = Color3.fromRGB(210, 25, 25),
	accentDim = Color3.fromRGB(60, 8, 8),
	text = Color3.fromRGB(210, 210, 210),
	textDim = Color3.fromRGB(115, 115, 115),
	green = Color3.fromRGB(40, 195, 75),
	red = Color3.fromRGB(210, 45, 45),
	white = Color3.fromRGB(255, 255, 255),
	black = Color3.fromRGB(0, 0, 0),
}
local FONT = Enum.Font.Gotham
local FONT_BOLD = Enum.Font.GothamBold
local MAIN_W = 212
local PAD = 8

local setFov, setDist, setPred, setTeamBtn, setWallBtn
local fovLbl, distLbl, predLbl, keybindBtn, statusLbl, dot
local setEspMaster, setEspHealth, setEspName, setEspDist
local tgtSubLbl

local function autoSave()
	local dataToSave = {
		savedConfigs = {},
		cfgCount = cfgCount,
		targetTeams = {},
		targetPlayers = {},
		espSettings = {
			enabled = esp.enabled,
			showHealth = esp.showHealth,
			showName = esp.showName,
			showDistance = esp.showDistance,
			fovEnabled = esp.fovEnabled, 
		},
		cfg = {
			lockSpeed = cfg.lockSpeed,
			fovRadius = cfg.fovRadius,
			maxDistance = cfg.maxDistance,
			prediction = cfg.prediction,
			teamCheck = cfg.teamCheck,
			wallCheck = cfg.wallCheck,
			keybind = cfg.keybind.Name,
		}
	}

	for teamName in pairs(esp.targetTeams) do
		table.insert(dataToSave.targetTeams, teamName)
	end

	for playerName in pairs(esp.targetPlayers) do
		table.insert(dataToSave.targetPlayers, playerName)
	end

	for name, config in pairs(savedConfigs) do
		local configData = {
			FOV = config.FOV,
			Distance = config.Distance,
			Prediction = config.Prediction,
			TeamCheck = config.TeamCheck,
			WallCheck = config.WallCheck,
			Keybind = config.Keybind,
			TargetTeams = {},
			TargetPlayers = {},
		}
		if config.TargetTeams then
			for _, team in ipairs(config.TargetTeams) do
				table.insert(configData.TargetTeams, team)
			end
		end
		if config.TargetPlayers then
			for _, playerName in ipairs(config.TargetPlayers) do
				table.insert(configData.TargetPlayers, playerName)
			end
		end
		dataToSave.savedConfigs[name] = configData
	end

	saveToLocalStorage(dataToSave)
end

local function loadData()
	local data = loadFromLocalStorage()
	if not data or not next(data) then
		return false
	end

	if data.cfg then
		cfg.fovRadius = data.cfg.fovRadius or 180
		cfg.maxDistance = data.cfg.maxDistance or 300
		cfg.prediction = data.cfg.prediction or 0.07
		cfg.teamCheck = data.cfg.teamCheck or false
		cfg.wallCheck = data.cfg.wallCheck or false
		if data.cfg.keybind then
			cfg.keybind = Enum.KeyCode[data.cfg.keybind] or Enum.KeyCode.Q
		end
	end

	if data.espSettings then
		esp.enabled = data.espSettings.enabled or false
		esp.showHealth = data.espSettings.showHealth or true
		esp.showName = data.espSettings.showName or true
		esp.showDistance = data.espSettings.showDistance or true
		esp.fovEnabled = data.espSettings.fovEnabled or true
	end

	if data.targetTeams then
		esp.targetTeams = {}
		for _, teamName in ipairs(data.targetTeams) do
			esp.targetTeams[teamName] = true
		end
	end

	if data.targetPlayers then
		esp.targetPlayers = {}
		for _, playerName in ipairs(data.targetPlayers) do
			esp.targetPlayers[playerName] = true
		end
	end

	if data.savedConfigs then
		savedConfigs = {}
		for name, configData in pairs(data.savedConfigs) do
			savedConfigs[name] = {
				FOV = configData.FOV,
				Distance = configData.Distance,
				Prediction = configData.Prediction,
				TeamCheck = configData.TeamCheck,
				WallCheck = configData.WallCheck,
				Keybind = configData.Keybind,
				TargetTeams = configData.TargetTeams,
				TargetPlayers = configData.TargetPlayers,
			}
		end
	end

	cfgCount = data.cfgCount or 0
	return true
end

local function applyConfig(c)
	if not c then return end

	cfg.fovRadius = c.FOV or cfg.fovRadius
	cfg.maxDistance = c.Distance or cfg.maxDistance
	cfg.teamCheck = c.TeamCheck or false
	cfg.wallCheck = c.WallCheck or false
	if c.Keybind then
		cfg.keybind = Enum.KeyCode[c.Keybind] or Enum.KeyCode.Q
	end

	if c.TargetTeams then
		esp.targetTeams = {}
		for _, team in ipairs(c.TargetTeams) do
			esp.targetTeams[team] = true
		end
	else
		esp.targetTeams = {}
	end

	if c.TargetPlayers then
		esp.targetPlayers = {}
		for _, playerName in ipairs(c.TargetPlayers) do
			esp.targetPlayers[playerName] = true
		end
	else
		esp.targetPlayers = {}
	end

	if setFov then setFov(cfg.fovRadius) end
	if setDist then setDist(cfg.maxDistance) end
	if setPred then setPred(cfg.prediction) end
	if setTeamBtn then setTeamBtn(cfg.teamCheck) end
	if setWallBtn then setWallBtn(cfg.wallCheck) end
	if fovLbl then fovLbl.Text = "FOV Radius: " .. math.floor(cfg.fovRadius) end
	if distLbl then distLbl.Text = "Max Distance: " .. math.floor(cfg.maxDistance) end
	if predLbl then predLbl.Text = "Prediction: " .. cfg.prediction .. "s" end
	if keybindBtn then keybindBtn.Text = "Keybind: " .. cfg.keybind.Name end
	if statusLbl then statusLbl.Text = (cfg.locked and "Lock: ON" or "Lock: OFF") .. " Bind: " .. cfg.keybind.Name end

	if tgtSubLbl then
		local teamCount = 0
		local playerCount = 0
		for _ in pairs(esp.targetTeams) do teamCount = teamCount + 1 end
		for _ in pairs(esp.targetPlayers) do playerCount = playerCount + 1 end
		if playerCount > 0 then
			tgtSubLbl.Text = "Players: " .. playerCount .. " selected"
		elseif teamCount > 0 then
			tgtSubLbl.Text = "Teams: " .. teamCount .. " selected"
		else
			tgtSubLbl.Text = "Selected: All Enemies"
		end
	end

	target = nil
	autoSave()
end

local function addCorner(p, r)
	local c = Instance.new("UICorner", p)
	c.CornerRadius = UDim.new(0, r or 4)
end

local function addStroke(p, col, thick)
	local s = Instance.new("UIStroke", p)
	s.Color = col or C.accent
	s.Thickness = thick or 1
end

local function makeBtn(parent, text, fs)
	local b = Instance.new("TextButton", parent)
	b.BackgroundColor3 = C.surface
	b.Text = text
	b.TextSize = fs or 11
	b.Font = FONT
	b.TextColor3 = C.text
	b.BorderSizePixel = 0
	b.AutoButtonColor = false
	addCorner(b, 3)
	addStroke(b, C.accentDim)
	b.MouseEnter:Connect(function()
		if b.BackgroundColor3 ~= C.red then
			b.BackgroundColor3 = C.accentDim
		end
	end)
	b.MouseLeave:Connect(function()
		if b.BackgroundColor3 ~= C.red then
			b.BackgroundColor3 = C.surface
		end
	end)
	return b
end

local function makeDraggable(frame, handle)
	local active, startMouse, startFrame = false, Vector2.new(), UDim2.new()
	handle.InputBegan:Connect(function(inp)
		if inp.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
		active = true
		startMouse = Vector2.new(inp.Position.X, inp.Position.Y)
		startFrame = frame.Position
	end)
	UserInputService.InputChanged:Connect(function(inp)
		if not active or inp.UserInputType ~= Enum.UserInputType.MouseMovement then return end
		local dx = inp.Position.X - startMouse.X
		local dy = inp.Position.Y - startMouse.Y
		frame.Position = UDim2.new(startFrame.X.Scale, startFrame.X.Offset + dx,
			startFrame.Y.Scale, startFrame.Y.Offset + dy)
	end)
	UserInputService.InputEnded:Connect(function(inp)
		if inp.UserInputType == Enum.UserInputType.MouseButton1 then active = false end
	end)
end

local function makeTitleBar(parent, text)
	local bar = Instance.new("Frame", parent)
	bar.Name = "TitleBar"
	bar.Size = UDim2.new(1, 0, 0, 26)
	bar.BackgroundColor3 = C.accentDim
	bar.BorderSizePixel = 0
	addCorner(bar, 5)
	local fix = Instance.new("Frame", bar)
	fix.Size = UDim2.new(1, 0, 0.5, 1)
	fix.Position = UDim2.new(0, 0, 0.5, 0)
	fix.BackgroundColor3 = C.accentDim
	fix.BorderSizePixel = 0
	local lbl = Instance.new("TextLabel", bar)
	lbl.Size = UDim2.new(1, 0, 1, 0)
	lbl.BackgroundTransparency = 1
	lbl.Text = text
	lbl.Font = FONT_BOLD
	lbl.TextSize = 12
	lbl.TextColor3 = C.white
	lbl.TextXAlignment = Enum.TextXAlignment.Center
	return bar
end

local function makeSeparator(parent, yPos)
	local s = Instance.new("Frame", parent)
	s.Size = UDim2.new(1, -(PAD * 2), 0, 1)
	s.Position = UDim2.new(0, PAD, 0, yPos)
	s.BackgroundColor3 = Color3.fromRGB(32, 32, 32)
	s.BorderSizePixel = 0
end

local function makeToggle(parent, label, initState, xOff, yOff, w, h, onChanged)
	local btn = makeBtn(parent, label .. ": " .. (initState and "ON" or "OFF"), 10)
	btn.Size = UDim2.new(0, w, 0, h)
	btn.Position = UDim2.new(0, xOff, 0, yOff)
	btn.TextColor3 = initState and C.green or C.text
	btn.MouseButton1Click:Connect(function()
		initState = not initState
		btn.Text = label .. ": " .. (initState and "ON" or "OFF")
		btn.TextColor3 = initState and C.green or C.text
		onChanged(initState)
		autoSave()
	end)
	local function set(v)
		initState = v
		btn.Text = label .. ": " .. (v and "ON" or "OFF")
		btn.TextColor3 = v and C.green or C.text
		onChanged(v)
	end
	return btn, set
end

local function makeSlider(parent, yOff, minV, maxV, initV, onChange)
	local track = Instance.new("Frame", parent)
	track.Size = UDim2.new(1, -(PAD * 2), 0, 6)
	track.Position = UDim2.new(0, PAD, 0, yOff)
	track.BackgroundColor3 = Color3.fromRGB(28, 28, 28)
	track.BorderSizePixel = 0
	addCorner(track, 3)
	addStroke(track, C.accentDim)

	local fill = Instance.new("Frame", track)
	fill.BackgroundColor3 = C.accent
	fill.BorderSizePixel = 0
	addCorner(fill, 3)

	local knob = Instance.new("Frame", track)
	knob.Size = UDim2.new(0, 10, 0, 14)
	knob.AnchorPoint = Vector2.new(0.5, 0.5)
	knob.BackgroundColor3 = Color3.fromRGB(220, 220, 220)
	knob.BorderSizePixel = 0
	addCorner(knob, 5)

	local function applyRel(rel)
		knob.Position = UDim2.new(rel, 0, 0.5, 0)
		fill.Size = UDim2.new(rel, 0, 1, 0)
	end
	applyRel((initV - minV) / (maxV - minV))

	local sliding = false
	local function fromX(x)
		local rel = math.clamp((x - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
		applyRel(rel)
		onChange(minV + (maxV - minV) * rel)
	end

	track.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 then sliding = true; fromX(i.Position.X) end
	end)
	UserInputService.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 then sliding = false end
	end)
	UserInputService.InputChanged:Connect(function(i)
		if sliding and i.UserInputType == Enum.UserInputType.MouseMovement then fromX(i.Position.X) end
	end)

	return function(v)
		applyRel(math.clamp((v - minV) / (maxV - minV), 0, 1))
		onChange(v)
		autoSave()
	end
end

local mainFrame = Instance.new("Frame", screenGui)
mainFrame.Name = "MainPanel"
mainFrame.Size = UDim2.new(0, MAIN_W, 0, 10)
mainFrame.Position = UDim2.new(0.02, 0, 0.18, 0)
mainFrame.BackgroundColor3 = C.bg
mainFrame.BorderSizePixel = 0
mainFrame.ClipsDescendants = true
addCorner(mainFrame, 5)
addStroke(mainFrame, C.accent, 1.5)

local mainBar = makeTitleBar(mainFrame, "RAIDEN")
makeDraggable(mainFrame, mainBar)

local CY = 30
local function nextY(h, gap) local y = CY; CY = CY + h + (gap or 4); return y end

local function rowLabel(text)
	local l = Instance.new("TextLabel", mainFrame)
	l.Size = UDim2.new(1, -(PAD * 2), 0, 13)
	l.Position = UDim2.new(0, PAD, 0, nextY(13, 2))
	l.BackgroundTransparency = 1
	l.Text = text
	l.Font = FONT
	l.TextSize = 10
	l.TextColor3 = C.textDim
	l.TextXAlignment = Enum.TextXAlignment.Left
	return l
end

local statRow = Instance.new("Frame", mainFrame)
statRow.Size = UDim2.new(1, -(PAD * 2), 0, 16)
statRow.Position = UDim2.new(0, PAD, 0, nextY(16, 5))
statRow.BackgroundTransparency = 1

dot = Instance.new("Frame", statRow)
dot.Size = UDim2.new(0, 7, 0, 7)
dot.Position = UDim2.new(0, 0, 0.5, -3)
dot.BackgroundColor3 = C.red
dot.BorderSizePixel = 0
addCorner(dot, 4)

statusLbl = Instance.new("TextLabel", statRow)
statusLbl.Size = UDim2.new(1, -12, 1, 0)
statusLbl.Position = UDim2.new(0, 12, 0, 0)
statusLbl.BackgroundTransparency = 1
statusLbl.Text = "Lock: OFF Bind: " .. cfg.keybind.Name
statusLbl.Font = FONT
statusLbl.TextSize = 10
statusLbl.TextColor3 = C.textDim
statusLbl.TextXAlignment = Enum.TextXAlignment.Left

makeSeparator(mainFrame, nextY(1, 5))

fovLbl = rowLabel("FOV Radius: " .. math.floor(cfg.fovRadius))
setFov = makeSlider(mainFrame, nextY(8, 7), 50, 400, cfg.fovRadius, function(v)
	cfg.fovRadius = v
	fovLbl.Text = "FOV Radius: " .. math.floor(v)
end)

distLbl = rowLabel("Max Distance: " .. math.floor(cfg.maxDistance))
setDist = makeSlider(mainFrame, nextY(8, 7), 50, 500, cfg.maxDistance, function(v)
	cfg.maxDistance = v
	distLbl.Text = "Max Distance: " .. math.floor(v)
end)

predLbl = rowLabel("Prediction: " .. cfg.prediction .. "s")
setPred = makeSlider(mainFrame, nextY(8, 7), 0, 0.3, cfg.prediction, function(v)
	cfg.prediction = math.floor(v * 1000) / 1000
	predLbl.Text = "Prediction: " .. cfg.prediction .. "s"
end)

makeSeparator(mainFrame, nextY(1, 5))

local twY = nextY(20, 4)

-- Define button width and padding for this row to ensure they fit
-- Adjusted button widths to ensure text fits within buttons, especially "OFF" state
local teamCheckBtnWidth = 70 -- Increased width for "Team Check"
local fovBtnWidth = 55       -- Increased width for "FOV"
local wallCheckBtnWidth = 70 -- Increased width for "Wall Check"
local buttonGap = 4

-- Calculate total width needed for buttons and gaps
local totalButtonWidth = teamCheckBtnWidth + fovBtnWidth + wallCheckBtnWidth + (buttonGap * 2)
local availableWidth = MAIN_W - (PAD * 2) -- Available width within the main frame, accounting for padding

-- Calculate the starting X position to center the group of buttons
local startX = PAD + (availableWidth - totalButtonWidth) / 2

local teamBtn
teamBtn, setTeamBtn = makeToggle(mainFrame, "Team Check", cfg.teamCheck, startX, twY, teamCheckBtnWidth, 20, function(v) cfg.teamCheck = v end)
teamBtn.TextXAlignment = Enum.TextXAlignment.Center -- Ensure text is centered

local fovToggleBtn
fovToggleBtn, setFovToggle = makeToggle(mainFrame, "FOV", esp.fovEnabled, startX + teamCheckBtnWidth + buttonGap, twY, fovBtnWidth, 20, function(v) esp.fovEnabled = v end)
fovToggleBtn.TextXAlignment = Enum.TextXAlignment.Center -- Ensure text is centered

local wallBtn
wallBtn, setWallBtn = makeToggle(mainFrame, "Wall Check", cfg.wallCheck, startX + teamCheckBtnWidth + buttonGap + fovBtnWidth + buttonGap, twY, wallCheckBtnWidth, 20, function(v) cfg.wallCheck = v end)
wallBtn.TextXAlignment = Enum.TextXAlignment.Center -- Ensure text is centered

keybindBtn = makeBtn(mainFrame, "Keybind: " .. cfg.keybind.Name, 11)
keybindBtn.Size = UDim2.new(1, -(PAD * 2), 0, 20)
keybindBtn.Position = UDim2.new(0, PAD, 0, nextY(20, 4))

makeSeparator(mainFrame, nextY(1, 5))

local btnRowY = nextY(20, 4)
local openEspBtn = makeBtn(mainFrame, "ESP", 11)
openEspBtn.Size = UDim2.new(0, 60, 0, 20)
openEspBtn.Position = UDim2.new(0, PAD, 0, btnRowY)

local openCfgBtn = makeBtn(mainFrame, "Configs", 11)
openCfgBtn.Size = UDim2.new(0, 70, 0, 20)
openCfgBtn.Position = UDim2.new(0, PAD + 64, 0, btnRowY)

local openTgtBtn = makeBtn(mainFrame, "Target", 11)
openTgtBtn.Size = UDim2.new(0, 60, 0, 20)
openTgtBtn.Position = UDim2.new(0, PAD + 138, 0, btnRowY)

mainFrame.Size = UDim2.new(0, MAIN_W, 0, CY + 6)

local espPanel = Instance.new("Frame", screenGui)
espPanel.Name = "EspPanel"
espPanel.Size = UDim2.new(0, 180, 0, 140)
espPanel.Position = UDim2.new(0.02, MAIN_W + 10, 0.18, 0)
espPanel.BackgroundColor3 = C.bg
espPanel.BorderSizePixel = 0
espPanel.Visible = false
addCorner(espPanel, 5)
addStroke(espPanel, C.accent, 1.5)

local espBar = makeTitleBar(espPanel, "ESP Settings")
makeDraggable(espPanel, espBar)

local EP_Y = 32
local function espNextY(h, g) local y = EP_Y; EP_Y = EP_Y + h + (g or 4); return y end

local espMasterBtn
espMasterBtn, setEspMaster = makeToggle(espPanel, "ESP", esp.enabled, PAD, espNextY(20, 4), 160, 20, function(v) esp.enabled = v end)

local espHealthBtn
espHealthBtn, setEspHealth = makeToggle(espPanel, "Health Bar", esp.showHealth, PAD, espNextY(20, 4), 160, 20, function(v) esp.showHealth = v end)

local espNameBtn
espNameBtn, setEspName = makeToggle(espPanel, "Name", esp.showName, PAD, espNextY(20, 4), 160, 20, function(v) esp.showName = v end)

local espDistBtn
espDistBtn, setEspDist = makeToggle(espPanel, "Distance", esp.showDistance, PAD, espNextY(20, 4), 160, 20, function(v) esp.showDistance = v end)

espPanel.Size = UDim2.new(0, 180, 0, EP_Y + 8)

local tgtPanel = Instance.new("Frame", screenGui)
tgtPanel.Name = "TargetPanel"
tgtPanel.Size = UDim2.new(0, 280, 0, 120)
tgtPanel.Position = UDim2.new(0.02, MAIN_W + 200, 0.18, 0)
tgtPanel.BackgroundColor3 = C.bg
tgtPanel.BorderSizePixel = 0
tgtPanel.Visible = false
addCorner(tgtPanel, 5)
addStroke(tgtPanel, C.accent, 1.5)

local tgtBar = makeTitleBar(tgtPanel, "Target Selection")
makeDraggable(tgtPanel, tgtBar)

local tabFrame = Instance.new("Frame", tgtPanel)
tabFrame.Size = UDim2.new(1, 0, 0, 24)
tabFrame.Position = UDim2.new(0, 0, 0, 28)
tabFrame.BackgroundTransparency = 1

local teamsTab = makeBtn(tabFrame, "Teams", 10)
teamsTab.Size = UDim2.new(0, 80, 1, 0)
teamsTab.Position = UDim2.new(0, PAD, 0, 0)

local playersTab = makeBtn(tabFrame, "Players", 10)
playersTab.Size = UDim2.new(0, 80, 1, 0)
playersTab.Position = UDim2.new(0, PAD + 85, 0, 0)

local activeTab = "teams"

tgtSubLbl = Instance.new("TextLabel", tgtPanel)
tgtSubLbl.Size = UDim2.new(1, -(PAD * 2), 0, 14)
tgtSubLbl.Position = UDim2.new(0, PAD, 0, 56)
tgtSubLbl.BackgroundTransparency = 1
tgtSubLbl.Text = "Selected: All Enemies"
tgtSubLbl.Font = FONT
tgtSubLbl.TextSize = 10
tgtSubLbl.TextColor3 = C.textDim
tgtSubLbl.TextXAlignment = Enum.TextXAlignment.Left

local function updateSubtitle()
	local teamCount = 0
	local playerCount = 0
	for _ in pairs(esp.targetTeams) do teamCount = teamCount + 1 end
	for _ in pairs(esp.targetPlayers) do playerCount = playerCount + 1 end

	if playerCount > 0 then
		local names = {}
		for name in pairs(esp.targetPlayers) do
			table.insert(names, name)
		end
		local display = table.concat(names, ", ")
		if string.len(display) > 35 then
			tgtSubLbl.Text = "Players: " .. playerCount .. " selected"
		else
			tgtSubLbl.Text = "Players: " .. display
		end
	elseif teamCount > 0 then
		local teams = {}
		for name in pairs(esp.targetTeams) do
			table.insert(teams, name)
		end
		local display = table.concat(teams, ", ")
		if string.len(display) > 35 then
			tgtSubLbl.Text = "Teams: " .. teamCount .. " selected"
		else
			tgtSubLbl.Text = "Teams: " .. display
		end
	else
		tgtSubLbl.Text = "Selected: All Enemies"
	end
end

local clearAllBtn = makeBtn(tgtPanel, "Clear All", 10)
clearAllBtn.Size = UDim2.new(1, -(PAD * 2), 0, 20)
clearAllBtn.Position = UDim2.new(0, PAD, 0, 74)
clearAllBtn.MouseButton1Click:Connect(function()
	esp.targetTeams = {}
	esp.targetPlayers = {}
	target = nil
	updateSubtitle()
	autoSave()
	clearAllBtn.TextColor3 = C.green
	task.delay(0.8, function() clearAllBtn.TextColor3 = C.text end)
	if activeTab == "teams" then rebuildTeamRows() else rebuildPlayerRows() end
end)

local teamScroll = Instance.new("ScrollingFrame", tgtPanel)
teamScroll.Size = UDim2.new(1, 0, 1, -104)
teamScroll.Position = UDim2.new(0, 0, 0, 100)
teamScroll.BackgroundTransparency = 1
teamScroll.BorderSizePixel = 0
teamScroll.ScrollBarThickness = 3
teamScroll.ScrollBarImageColor3 = C.accent
teamScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
teamScroll.Visible = true

local playerScroll = Instance.new("ScrollingFrame", tgtPanel)
playerScroll.Size = UDim2.new(1, 0, 1, -104)
playerScroll.Position = UDim2.new(0, 0, 0, 100)
playerScroll.BackgroundTransparency = 1
playerScroll.BorderSizePixel = 0
playerScroll.ScrollBarThickness = 3
playerScroll.ScrollBarImageColor3 = C.accent
playerScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
playerScroll.Visible = false

local teamList = Instance.new("UIListLayout", teamScroll)
teamList.Padding = UDim.new(0, 3)
teamList.SortOrder = Enum.SortOrder.Name

local playerList = Instance.new("UIListLayout", playerScroll)
playerList.Padding = UDim.new(0, 3)
playerList.SortOrder = Enum.SortOrder.Name

local function getTeamColor(teamName)
	local teams = TeamsService:GetTeams()
	for _, t in ipairs(teams) do
		if t.Name == teamName then
			return t.TeamColor.Color
		end
	end
	return C.textDim
end

local function rebuildTeamRows()
	for _, c in ipairs(teamScroll:GetChildren()) do
		if c:IsA("TextButton") then c:Destroy() end
	end

	local seen = {}
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr ~= player and plr.Team then
			local name = plr.Team.Name
			if not seen[name] then
				seen[name] = true
				local col = getTeamColor(name)
				local isSelected = esp.targetTeams[name] == true

				local row = makeBtn(teamScroll, name, 10)
				row.Size = UDim2.new(1, -(PAD * 2), 0, 22)
				row.BackgroundColor3 = isSelected and C.red or C.surface
				row.TextColor3 = isSelected and C.white or col
				row.TextXAlignment = Enum.TextXAlignment.Left
				row.AutoButtonColor = false

				local indicator = Instance.new("TextLabel", row)
				indicator.Name = "Indicator"
				indicator.Size = UDim2.new(0, 20, 1, 0)
				indicator.Position = UDim2.new(1, -22, 0, 0)
				indicator.BackgroundTransparency = 1
				indicator.Text = isSelected and "[X]" or "[ ]"
				indicator.TextColor3 = isSelected and C.white or C.textDim
				indicator.Font = FONT_BOLD
				indicator.TextSize = 12
				indicator.TextXAlignment = Enum.TextXAlignment.Center

				row.MouseEnter:Connect(function()
					if row.BackgroundColor3 ~= C.red then
						row.BackgroundColor3 = C.accentDim
						indicator.TextColor3 = C.text
					end
				end)
				row.MouseLeave:Connect(function()
					if row.BackgroundColor3 == C.accentDim then
						row.BackgroundColor3 = isSelected and C.red or C.surface
						indicator.TextColor3 = isSelected and C.white or C.textDim
					end
				end)

				local n = name
				row.MouseButton1Click:Connect(function()
					if esp.targetTeams[n] then
						esp.targetTeams[n] = nil
					else
						esp.targetTeams[n] = true
						esp.targetPlayers = {}
					end
					target = nil
					updateSubtitle()
					autoSave()
					rebuildTeamRows()
					rebuildPlayerRows()
				end)
			end
		end
	end

	local count = 0
	for _, c in ipairs(teamScroll:GetChildren()) do
		if c:IsA("TextButton") then count = count + 1 end
	end
	teamScroll.CanvasSize = UDim2.new(0, 0, 0, count * 25)
end

local function rebuildPlayerRows()
	for _, c in ipairs(playerScroll:GetChildren()) do
		if c:IsA("TextButton") then c:Destroy() end
	end

	for _, plr in ipairs(Players:GetPlayers()) do
		if plr ~= player then
			local name = plr.Name
			local displayName = plr.DisplayName
			local teamName = plr.Team and plr.Team.Name or "No Team"
			local isSelected = esp.targetPlayers[name] == true
			local teamCol = plr.Team and plr.TeamColor.Color or C.white

			local row = makeBtn(playerScroll, displayName .. " (" .. teamName .. ")", 10)
			row.Size = UDim2.new(1, -(PAD * 2), 0, 22)
			row.BackgroundColor3 = isSelected and C.red or C.surface
			row.TextColor3 = isSelected and C.white or teamCol
			row.TextXAlignment = Enum.TextXAlignment.Left
			row.AutoButtonColor = false

			local indicator = Instance.new("TextLabel", row)
			indicator.Name = "Indicator"
			indicator.Size = UDim2.new(0, 20, 1, 0)
			indicator.Position = UDim2.new(1, -22, 0, 0)
			indicator.BackgroundTransparency = 1
			indicator.Text = isSelected and "[X]" or "[ ]"
			indicator.TextColor3 = isSelected and C.white or C.textDim
			indicator.Font = FONT_BOLD
			indicator.TextSize = 12
			indicator.TextXAlignment = Enum.TextXAlignment.Center

			row.MouseEnter:Connect(function()
				if row.BackgroundColor3 ~= C.red then
					row.BackgroundColor3 = C.accentDim
					indicator.TextColor3 = C.text
				end
			end)
			row.MouseLeave:Connect(function()
				if row.BackgroundColor3 == C.accentDim then
					row.BackgroundColor3 = isSelected and C.red or C.surface
					indicator.TextColor3 = isSelected and C.white or C.textDim
				end
			end)

			local n = name
			row.MouseButton1Click:Connect(function()
				if esp.targetPlayers[n] then
					esp.targetPlayers[n] = nil
				else
					esp.targetPlayers[n] = true
					esp.targetTeams = {}
				end
				target = nil
				updateSubtitle()
				autoSave()
				rebuildTeamRows()
				rebuildPlayerRows()
			end)
		end
	end

	local count = 0
	for _, c in ipairs(playerScroll:GetChildren()) do
		if c:IsA("TextButton") then count = count + 1 end
	end
	playerScroll.CanvasSize = UDim2.new(0, 0, 0, count * 25)

	local newHeight = math.clamp(104 + count * 25 + 8, 120, 380)
	tgtPanel.Size = UDim2.new(0, 280, 0, newHeight)
end

teamsTab.MouseButton1Click:Connect(function()
	activeTab = "teams"
	teamScroll.Visible = true
	playerScroll.Visible = false
	teamsTab.BackgroundColor3 = C.accentDim
	playersTab.BackgroundColor3 = C.surface
	rebuildTeamRows()
end)

playersTab.MouseButton1Click:Connect(function()
	activeTab = "players"
	teamScroll.Visible = false
	playerScroll.Visible = true
	teamsTab.BackgroundColor3 = C.surface
	playersTab.BackgroundColor3 = C.accentDim
	rebuildPlayerRows()
end)

teamsTab.BackgroundColor3 = C.accentDim

openTgtBtn.MouseButton1Click:Connect(function()
	tgtPanel.Visible = not tgtPanel.Visible
	if tgtPanel.Visible then
		if activeTab == "teams" then rebuildTeamRows() else rebuildPlayerRows() end
	end
end)

local cfgPanel = Instance.new("Frame", screenGui)
cfgPanel.Name = "ConfigPanel"
cfgPanel.Size = UDim2.new(0, 280, 0, 80)
cfgPanel.Position = UDim2.new(0.02, MAIN_W + 10, 0.38, 0)
cfgPanel.BackgroundColor3 = C.bg
cfgPanel.BorderSizePixel = 0
cfgPanel.Visible = false
addCorner(cfgPanel, 5)
addStroke(cfgPanel, C.accent, 1.5)

local cfgBar = makeTitleBar(cfgPanel, "Config Manager")
makeDraggable(cfgPanel, cfgBar)

local saveCfgBtn = makeBtn(cfgPanel, "Save Current Config", 11)
saveCfgBtn.Size = UDim2.new(0, 120, 0, 20)
saveCfgBtn.Position = UDim2.new(0, PAD, 0, 32)

local exportBtn = makeBtn(cfgPanel, "Export Code", 11)
exportBtn.Size = UDim2.new(0, 80, 0, 20)
exportBtn.Position = UDim2.new(0, 128, 0, 32)

local importBtn = makeBtn(cfgPanel, "Import Code", 11)
importBtn.Size = UDim2.new(0, 50, 0, 20)
importBtn.Position = UDim2.new(0, 216, 0, 32)

local CFG_ROW_H = 22
local CFG_ROW_GAP = 3
local CFG_MAX_VIS = 5

local cfgScroll = Instance.new("ScrollingFrame", cfgPanel)
cfgScroll.Size = UDim2.new(1, 0, 0, 0)
cfgScroll.Position = UDim2.new(0, 0, 0, 58)
cfgScroll.BackgroundTransparency = 1
cfgScroll.BorderSizePixel = 0
cfgScroll.ScrollBarThickness = 3
cfgScroll.ScrollBarImageColor3 = C.accent
cfgScroll.CanvasSize = UDim2.new(0, 0, 0, 0)

local cfgList = Instance.new("UIListLayout", cfgScroll)
cfgList.Padding = UDim.new(0, CFG_ROW_GAP)
cfgList.SortOrder = Enum.SortOrder.Name

local cfgListPad = Instance.new("UIPadding", cfgScroll)
cfgListPad.PaddingLeft = UDim.new(0, PAD)
cfgListPad.PaddingRight = UDim.new(0, PAD)
cfgListPad.PaddingTop = UDim.new(0, 2)

-- Notification inside Config Panel
local cfgNotif = Instance.new("TextLabel", cfgPanel)
cfgNotif.Size = UDim2.new(1, -(PAD * 2), 0, 18)
cfgNotif.Position = UDim2.new(0, PAD, 0, 32)
cfgNotif.BackgroundColor3 = Color3.fromRGB(20, 60, 20)
cfgNotif.BorderSizePixel = 0
cfgNotif.Font = FONT
cfgNotif.TextSize = 13
cfgNotif.TextScaled = false
cfgNotif.TextColor3 = C.green
cfgNotif.Text = ""
cfgNotif.Visible = false
cfgNotif.ZIndex = 15
addCorner(cfgNotif, 3)
addStroke(cfgNotif, C.accentDim)

local function showNotif(msg, isError)
	cfgNotif.Text = msg
	cfgNotif.TextColor3 = isError and C.red or C.green
	cfgNotif.BackgroundColor3 = isError and Color3.fromRGB(60, 20, 20) or Color3.fromRGB(20, 60, 20)
	cfgNotif.Visible = true
	cfgNotif.TextTransparency = 0
	cfgNotif.BackgroundTransparency = 0
	saveCfgBtn.Position = UDim2.new(0, PAD, 0, 54)
	exportBtn.Position = UDim2.new(0, 128, 0, 54)
	importBtn.Position = UDim2.new(0, 216, 0, 54)
	task.delay(2, function()
		for i = 1, 10 do
			cfgNotif.TextTransparency = i / 10
			cfgNotif.BackgroundTransparency = i / 10
			task.wait(0.04)
		end
		cfgNotif.Visible = false
		saveCfgBtn.Position = UDim2.new(0, PAD, 0, 32)
		exportBtn.Position = UDim2.new(0, 128, 0, 32)
		importBtn.Position = UDim2.new(0, 216, 0, 32)
	end)
end

local renameBox = Instance.new("TextBox", cfgPanel)
renameBox.BackgroundColor3 = C.surface
renameBox.TextColor3 = C.text
renameBox.PlaceholderColor3 = C.textDim
renameBox.Font = FONT
renameBox.TextSize = 11
renameBox.TextXAlignment = Enum.TextXAlignment.Left
renameBox.Visible = false
renameBox.ClearTextOnFocus = true
renameBox.PlaceholderText = "New name then Enter"
renameBox.BorderSizePixel = 0
renameBox.ClipsDescendants = true
addCorner(renameBox, 3)
addStroke(renameBox, C.accentDim)
local renamePad = Instance.new("UIPadding", renameBox)
renamePad.PaddingLeft = UDim.new(0, 6)
renamePad.PaddingRight = UDim.new(0, 6)
renamePad.PaddingTop = UDim.new(0, 1)

local importBox = Instance.new("TextBox", cfgPanel)
importBox.BackgroundColor3 = C.surface
importBox.TextColor3 = C.text
importBox.PlaceholderColor3 = C.textDim
importBox.Font = FONT
importBox.TextSize = 11
importBox.TextXAlignment = Enum.TextXAlignment.Left
importBox.Visible = false
importBox.ClearTextOnFocus = true
importBox.PlaceholderText = "Paste code here then Enter"
importBox.BorderSizePixel = 0
importBox.ClipsDescendants = true
addCorner(importBox, 3)
addStroke(importBox, C.accentDim)
local importPad = Instance.new("UIPadding", importBox)
importPad.PaddingLeft = UDim.new(0, 6)
importPad.PaddingRight = UDim.new(0, 6)
importPad.PaddingTop = UDim.new(0, 1)

-- Yes/No Delete Confirmation (centered inside Config Panel)
local confirmFrame = Instance.new("Frame", cfgPanel)
confirmFrame.BackgroundColor3 = C.bg
confirmFrame.BorderSizePixel = 0
confirmFrame.Visible = false
confirmFrame.ZIndex = 20
addStroke(confirmFrame, C.accent, 1.5)
addCorner(confirmFrame, 5)

local confirmLbl = Instance.new("TextLabel", confirmFrame)
confirmLbl.Size = UDim2.new(1, 0, 0, 20)
confirmLbl.Position = UDim2.new(0, 0, 0, 6)
confirmLbl.BackgroundTransparency = 1
confirmLbl.Font = FONT_BOLD
confirmLbl.TextSize = 10
confirmLbl.TextColor3 = C.text
confirmLbl.Text = "Delete this config?"
confirmLbl.ZIndex = 21

local confirmYes = makeBtn(confirmFrame, "Yes", 10)
confirmYes.Size = UDim2.new(0, 70, 0, 18)
confirmYes.Position = UDim2.new(0, 8, 0, 30)
confirmYes.TextColor3 = C.green
confirmYes.ZIndex = 21

local confirmNo = makeBtn(confirmFrame, "No", 10)
confirmNo.Size = UDim2.new(0, 70, 0, 18)
confirmNo.Position = UDim2.new(0, 84, 0, 30)
confirmNo.TextColor3 = C.red
confirmNo.ZIndex = 21

local pendingDelete = nil
local pendingRename = nil

local rebuildCfgRows

local function hideConfirm()
	confirmFrame.Visible = false
	pendingDelete = nil
end

local function updatePanelHeight()
	local count = 0
	for _ in pairs(savedConfigs) do count = count + 1 end
	local visH = math.min(count, CFG_MAX_VIS) * (CFG_ROW_H + CFG_ROW_GAP)
	cfgScroll.Size = UDim2.new(1, 0, 0, visH)
	cfgScroll.ScrollBarThickness = (count > CFG_MAX_VIS) and 3 or 0
	cfgPanel.Size = UDim2.new(0, 280, 0, math.max(80, 58 + visH + 10))
end

confirmNo.MouseButton1Click:Connect(hideConfirm)

confirmYes.MouseButton1Click:Connect(function()
	if pendingDelete then
		local deletedName = pendingDelete
		savedConfigs[pendingDelete] = nil
		hideConfirm()
		autoSave()
		rebuildCfgRows()
		showNotif("Deleted: " .. deletedName)
	end
end)

local function serializeTargetsForSave(targets)
	local arr = {}
	for key in pairs(targets) do table.insert(arr, key) end
	return arr
end

local function generateExportCode()
	local exportData = {
		version = 1,
		cfg = {
			fovRadius = cfg.fovRadius,
			maxDistance = cfg.maxDistance,
			prediction = cfg.prediction,
			teamCheck = cfg.teamCheck,
			wallCheck = cfg.wallCheck,
			keybind = cfg.keybind.Name,
		},
		esp = {
			enabled = esp.enabled,
			showHealth = esp.showHealth,
			showName = esp.showName,
			showDistance = esp.showDistance,
		},
		targetTeams = {},
		targetPlayers = {},
	}

	for teamName in pairs(esp.targetTeams) do
		table.insert(exportData.targetTeams, teamName)
	end

	for playerName in pairs(esp.targetPlayers) do
		table.insert(exportData.targetPlayers, playerName)
	end

	local json = HttpService:JSONEncode(exportData)
	local encoded = base64Encode(json)

	return encoded
end

local function importFromCode(code)
	if not code or code == "" then
		return false, "Empty code"
	end

	local decoded = base64Decode(code)
	if not decoded or decoded == "" then
		return false, "Invalid code format"
	end

	local success, importData = pcall(function()
		return HttpService:JSONDecode(decoded)
	end)
	if not success or not importData then
		return false, "Invalid data format"
	end

	if importData.version ~= 1 then
		return false, "Unsupported version"
	end

	if importData.cfg then
		if importData.cfg.fovRadius then cfg.fovRadius = importData.cfg.fovRadius end
		if importData.cfg.maxDistance then cfg.maxDistance = importData.cfg.maxDistance end
		if importData.cfg.prediction then cfg.prediction = importData.cfg.prediction end
		if importData.cfg.teamCheck ~= nil then cfg.teamCheck = importData.cfg.teamCheck end
		if importData.cfg.wallCheck ~= nil then cfg.wallCheck = importData.cfg.wallCheck end
		if importData.cfg.keybind then
			cfg.keybind = Enum.KeyCode[importData.cfg.keybind] or Enum.KeyCode.Q
		end
	end

	if importData.esp then
		if importData.esp.enabled ~= nil then esp.enabled = importData.esp.enabled end
		if importData.esp.showHealth ~= nil then esp.showHealth = importData.esp.showHealth end
		if importData.esp.showName ~= nil then esp.showName = importData.esp.showName end
		if importData.esp.showDistance ~= nil then esp.showDistance = importData.esp.showDistance end
	end

	if importData.targetTeams then
		esp.targetTeams = {}
		for _, teamName in ipairs(importData.targetTeams) do
			esp.targetTeams[teamName] = true
		end
	else
		esp.targetTeams = {}
	end

	if importData.targetPlayers then
		esp.targetPlayers = {}
		for _, playerName in ipairs(importData.targetPlayers) do
			esp.targetPlayers[playerName] = true
		end
	else
		esp.targetPlayers = {}
	end

	-- Update UI
	if setFov then setFov(cfg.fovRadius) end
	if setDist then setDist(cfg.maxDistance) end
	if setPred then setPred(cfg.prediction) end
	if setTeamBtn then setTeamBtn(cfg.teamCheck) end
	if setWallBtn then setWallBtn(cfg.wallCheck) end
	if fovLbl then fovLbl.Text = "FOV Radius: " .. math.floor(cfg.fovRadius) end
	if distLbl then distLbl.Text = "Max Distance: " .. math.floor(cfg.maxDistance) end
	if predLbl then predLbl.Text = "Prediction: " .. cfg.prediction .. "s" end
	if keybindBtn then keybindBtn.Text = "Keybind: " .. cfg.keybind.Name end
	if statusLbl then statusLbl.Text = (cfg.locked and "Lock: ON" or "Lock: OFF") .. " Bind: " .. cfg.keybind.Name end
	if setEspMaster then setEspMaster(esp.enabled) end
	if setEspHealth then setEspHealth(esp.showHealth) end
	if setEspName then setEspName(esp.showName) end
	if setEspDist then setEspDist(esp.showDistance) end

	updateSubtitle()
	autoSave()

	return true, "Import successful!"
end

exportBtn.MouseButton1Click:Connect(function()
	local code = generateExportCode()
	setclipboard(code)
	showNotif("Code copied to clipboard!")
end)

importBtn.MouseButton1Click:Connect(function()
	importBox.Size = UDim2.new(1, -(PAD * 2), 0, 20)
	importBox.Position = UDim2.new(0, PAD, 0, cfgPanel.Size.Y.Offset - 4)
	importBox.Text = ""
	importBox.Visible = true
	cfgPanel.Size = UDim2.new(0, 280, 0, cfgPanel.Size.Y.Offset + 26)
	importBox:CaptureFocus()
end)

importBox.FocusLost:Connect(function(enter)
	if enter and importBox.Text ~= "" then
		local success, message = importFromCode(importBox.Text)
		showNotif(message, not success)
		if success then
			rebuildTeamRows()
			rebuildPlayerRows()
			rebuildCfgRows()
		end
	end
	importBox.Visible = false
	updatePanelHeight()
end)

rebuildCfgRows = function()
	for _, child in ipairs(cfgScroll:GetChildren()) do
		if child:IsA("Frame") and child.Name == "CfgRow" then
			child:Destroy()
		end
	end

	local count = 0
	for _ in pairs(savedConfigs) do count = count + 1 end

	cfgScroll.CanvasSize = UDim2.new(0, 0, 0, count * (CFG_ROW_H + CFG_ROW_GAP))
	updatePanelHeight()

	for name in pairs(savedConfigs) do
		local row = Instance.new("Frame", cfgScroll)
		row.Name = "CfgRow"
		row.Size = UDim2.new(1, 0, 0, CFG_ROW_H)
		row.BackgroundTransparency = 1
		row.BorderSizePixel = 0

		local loadBtn = makeBtn(row, name, 10)
		loadBtn.Size = UDim2.new(0, 120, 1, 0)
		loadBtn.Position = UDim2.new(0, 0, 0, 0)
		loadBtn.TextTruncate = Enum.TextTruncate.AtEnd
		loadBtn.TextXAlignment = Enum.TextXAlignment.Left
		local n = name
		loadBtn.MouseButton1Click:Connect(function()
			if savedConfigs[n] then applyConfig(savedConfigs[n]) end
		end)

		local rnBtn = makeBtn(row, "Rename", 10)
		rnBtn.Size = UDim2.new(0, 50, 1, 0)
		rnBtn.Position = UDim2.new(0, 124, 0, 0)
		local rn = name
		rnBtn.MouseButton1Click:Connect(function()
			pendingRename = rn
			hideConfirm()
			renameBox.Size = UDim2.new(1, -(PAD * 2), 0, 20)
			renameBox.Position = UDim2.new(0, PAD, 0, cfgPanel.Size.Y.Offset - 4)
			renameBox.Text = ""
			renameBox.Visible = true
			cfgPanel.Size = UDim2.new(0, 280, 0, cfgPanel.Size.Y.Offset + 26)
			renameBox:CaptureFocus()
		end)

		local delBtn = makeBtn(row, "Del", 10)
		delBtn.Size = UDim2.new(0, 32, 1, 0)
		delBtn.Position = UDim2.new(0, 178, 0, 0)
		delBtn.TextColor3 = C.red
		local dn = name
		delBtn.MouseButton1Click:Connect(function()
			pendingDelete = dn
			renameBox.Visible = false
			importBox.Visible = false
			confirmFrame.Size = UDim2.new(0, 160, 0, 56)
			confirmFrame.Position = UDim2.new(0.5, -80, 0, cfgPanel.Size.Y.Offset - 70)
			confirmFrame.Visible = true
		end)

		local exportCfgBtn = makeBtn(row, "Code", 10)
		exportCfgBtn.Size = UDim2.new(0, 40, 1, 0)
		exportCfgBtn.Position = UDim2.new(0, 214, 0, 0)
		local cfgName = name
		exportCfgBtn.MouseButton1Click:Connect(function()
			local config = savedConfigs[cfgName]
			if config then
				local exportData = {
					version = 1,
					cfg = {
						fovRadius = config.FOV,
						maxDistance = config.Distance,
						prediction = config.Prediction,
						teamCheck = config.TeamCheck,
						wallCheck = config.WallCheck,
						keybind = config.Keybind,
					},
					targetTeams = config.TargetTeams or {},
					targetPlayers = config.TargetPlayers or {},
				}
				local json = HttpService:JSONEncode(exportData)
				local encoded = base64Encode(json)
				setclipboard(encoded)
				showNotif("Config code copied!")
			end
		end)
	end
end

renameBox.FocusLost:Connect(function(enter)
	if enter and pendingRename and renameBox.Text ~= "" and renameBox.Text ~= pendingRename then
		savedConfigs[renameBox.Text] = savedConfigs[pendingRename]
		savedConfigs[pendingRename] = nil
		autoSave()
		rebuildCfgRows()
	end
	renameBox.Visible = false
	pendingRename = nil
	updatePanelHeight()
end)

saveCfgBtn.MouseButton1Click:Connect(function()
	cfgCount = cfgCount + 1
	local configName = "Config " .. cfgCount
	savedConfigs[configName] = {
		FOV = cfg.fovRadius,
		Distance = cfg.maxDistance,
		Prediction = cfg.prediction,
		TeamCheck = cfg.teamCheck,
		WallCheck = cfg.wallCheck,
		Keybind = cfg.keybind.Name,
		TargetTeams = serializeTargetsForSave(esp.targetTeams),
		TargetPlayers = serializeTargetsForSave(esp.targetPlayers),
	}
	rebuildCfgRows()
	autoSave()
	saveCfgBtn.TextColor3 = C.green
	task.delay(1, function() saveCfgBtn.TextColor3 = C.text end)
	showNotif("Config saved: " .. configName)
end)

openEspBtn.MouseButton1Click:Connect(function() espPanel.Visible = not espPanel.Visible end)
openCfgBtn.MouseButton1Click:Connect(function()
	cfgPanel.Visible = not cfgPanel.Visible
	if cfgPanel.Visible then
		hideConfirm()
		rebuildCfgRows()
	end
end)

local waitKey = false
keybindBtn.MouseButton1Click:Connect(function()
	waitKey = true
	keybindBtn.Text = "Press any key..."
end)

UserInputService.InputBegan:Connect(function(inp, gp)
	if waitKey and not gp then
		waitKey = false
		if inp.KeyCode ~= Enum.KeyCode.Unknown then
			cfg.keybind = inp.KeyCode
			keybindBtn.Text = "Keybind: " .. inp.KeyCode.Name
			statusLbl.Text = (cfg.locked and "Lock: ON" or "Lock: OFF") .. " Bind: " .. inp.KeyCode.Name
			autoSave()
		end
		return
	end
	if gp then return end
	if inp.KeyCode == cfg.keybind then
		cfg.locked = not cfg.locked
		target = nil
		dot.BackgroundColor3 = cfg.locked and C.green or C.red
		statusLbl.Text = (cfg.locked and "Lock: ON" or "Lock: OFF") .. " Bind: " .. cfg.keybind.Name
		statusLbl.TextColor3 = cfg.locked and C.green or C.textDim
	end
end)

local fovCircle = Drawing.new("Circle")
fovCircle.Color = Color3.fromRGB(210, 25, 25)
fovCircle.Thickness = 1
fovCircle.NumSides = 64
fovCircle.Filled = false
fovCircle.Visible = true

local function newDrawing(dtype, props)
	local d = Drawing.new(dtype)
	for k, v in pairs(props) do d[k] = v end
	return d
end

local function newOutlinedText()
	local text = Drawing.new("Text")
	text.Size = 13
	text.Font = Drawing.Fonts.Plex
	text.Center = true
	text.Outline = true
	text.OutlineColor = Color3.fromRGB(0, 0, 0)
	text.Thickness = 2
	text.Visible = false
	return text
end

local function getOrCreateEsp(plr)
	if espDrawings[plr] then return espDrawings[plr] end

	local d = {
		box = newDrawing("Square", {
			Thickness = 1.5, Filled = false, Visible = false,
			Color = C.white,
		}),
		nameTag = newOutlinedText(),
		distTag = newOutlinedText(),
		healthBg = newDrawing("Square", {
			Thickness = 1, Filled = true, Visible = false,
			Color = Color3.fromRGB(0, 0, 0),
		}),
		healthFill = newDrawing("Square", {
			Thickness = 1, Filled = true, Visible = false,
			Color = C.green,
		}),
	}
	d.distTag.Size = 11
	espDrawings[plr] = d
	return d
end

local function hideEsp(d)
	d.box.Visible = false
	d.nameTag.Visible = false
	d.distTag.Visible = false
	d.healthBg.Visible = false
	d.healthFill.Visible = false
end

local function removeEsp(plr)
	local d = espDrawings[plr]
	if not d then return end
	for _, v in pairs(d) do v:Remove() end
	espDrawings[plr] = nil
end

Players.PlayerRemoving:Connect(removeEsp)

local function playerTeamColor(plr)
	if plr.Team then
		return plr.TeamColor.Color
	end
	return C.white
end

local function getCharBounds(char)
	local hrp = char:FindFirstChild("HumanoidRootPart")
	local head = char:FindFirstChild("Head")
	if not hrp or not head then return nil end

	local topPos, topVis = camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.7, 0))
	local bottomPos, bottomVis = camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3.0, 0))

	if not topVis and not bottomVis then return nil end
	if topPos.Z <= 0 then return nil end

	local height = math.abs(topPos.Y - bottomPos.Y)
	local width = height * 0.55
	local x = topPos.X - width / 2
	local y = topPos.Y
	return x, y, width, height, topPos.Z
end

local function aimPoint(char)
	local head = char:FindFirstChild("Head")
	local hrp = char:FindFirstChild("HumanoidRootPart")
	local base = head and head.Position or (hrp and hrp.Position)
	if not base or not hrp then return nil end
	return base + hrp.AssemblyLinearVelocity * cfg.prediction
end

local function isAlive(char)
	local h = char and char:FindFirstChildOfClass("Humanoid")
	return h and h.Health > 0
end

local function playerPassesFilter(plr)
	if plr == player then return false end
	local char = plr.Character
	if not char or not isAlive(char) then return false end
	if not char:FindFirstChild("HumanoidRootPart") then return false end

	if next(esp.targetPlayers) ~= nil then
		if not esp.targetPlayers[plr.Name] then return false end
	elseif next(esp.targetTeams) ~= nil then
		if not plr.Team or not esp.targetTeams[plr.Team.Name] then return false end
	else
		if cfg.teamCheck and plr.Team == player.Team then return false end
	end
	return true
end

local function getClosest()
	local best, bestDist = nil, math.huge
	local mouse = UserInputService:GetMouseLocation()
	local myChar = player.Character
	local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
	if not myHRP then return nil end

	for _, plr in ipairs(Players:GetPlayers()) do
		if not playerPassesFilter(plr) then continue end
		local char = plr.Character
		local hrp = char:FindFirstChild("HumanoidRootPart")
		if (hrp.Position - myHRP.Position).Magnitude > cfg.maxDistance then continue end

		local aim = aimPoint(char)
		if not aim then continue end
		local sp, on = camera:WorldToViewportPoint(aim)
		if not on then continue end

		local d2 = (Vector2.new(sp.X, sp.Y) - mouse).Magnitude
		if d2 > cfg.fovRadius then continue end

		if cfg.wallCheck then
			local rp = RaycastParams.new()
			rp.FilterDescendantsInstances = { myChar }
			rp.FilterType = Enum.RaycastFilterType.Blacklist
			local res = workspace:Raycast(camera.CFrame.Position, hrp.Position - camera.CFrame.Position, rp)
			if res and not char:IsAncestorOf(res.Instance) then continue end
		end

		if d2 < bestDist then bestDist = d2; best = plr end
	end
	return best
end

task.wait(1)
local loaded = loadData()

if setFov then setFov(cfg.fovRadius) end
if setDist then setDist(cfg.maxDistance) end
if setPred then setPred(cfg.prediction) end
if setTeamBtn then setTeamBtn(cfg.teamCheck) end
if setWallBtn then setWallBtn(cfg.wallCheck) end
if fovLbl then fovLbl.Text = "FOV Radius: " .. math.floor(cfg.fovRadius) end
if distLbl then distLbl.Text = "Max Distance: " .. math.floor(cfg.maxDistance) end
if predLbl then predLbl.Text = "Prediction: " .. cfg.prediction .. "s" end
if keybindBtn then keybindBtn.Text = "Keybind: " .. cfg.keybind.Name end
if statusLbl then statusLbl.Text = (cfg.locked and "Lock: ON" or "Lock: OFF") .. " Bind: " .. cfg.keybind.Name end
if setEspMaster then setEspMaster(esp.enabled) end
if setEspHealth then setEspHealth(esp.showHealth) end
if setEspName then setEspName(esp.showName) end
if setEspDist then setEspDist(esp.showDistance) end

updateSubtitle()
rebuildCfgRows()

RunService.RenderStepped:Connect(function(dt)
	fovCircle.Position = UserInputService:GetMouseLocation()
	fovCircle.Radius = cfg.fovRadius
	fovCircle.Visible = esp.fovEnabled

	local myChar = player.Character
	local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")

	-- ESP drawing logic (remains the same)
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr == player then continue end
		local d = getOrCreateEsp(plr)

		local char = plr.Character
		local shouldDraw = esp.enabled and char and isAlive(char)

		if shouldDraw then
			if next(esp.targetPlayers) ~= nil then
				shouldDraw = esp.targetPlayers[plr.Name] == true
			elseif next(esp.targetTeams) ~= nil then
				shouldDraw = plr.Team ~= nil and esp.targetTeams[plr.Team.Name] == true
			end
		end

		if not shouldDraw then hideEsp(d); continue end

		-- ADD THIS CHECK FOR FOV ENABLED
		if not esp.fovEnabled then
			hideEsp(d)
			continue
		end

		local x, y, w, h, depth = getCharBounds(char)
		if not x then hideEsp(d); continue end

		local teamCol = playerTeamColor(plr)

		d.box.Visible = true
		d.box.Color = teamCol
		d.box.Position = Vector2.new(x, y)
		d.box.Size = Vector2.new(w, h)

		if esp.showName then
			d.nameTag.Visible = true
			d.nameTag.Color = teamCol
			d.nameTag.Text = plr.DisplayName
			d.nameTag.Position = Vector2.new(x + w / 2, y - 14)
		else
			d.nameTag.Visible = false
		end

		if esp.showDistance and myHRP then
			local hrp = char:FindFirstChild("HumanoidRootPart")
			if hrp then
				local dist = math.floor((hrp.Position - myHRP.Position).Magnitude)
				d.distTag.Visible = true
				d.distTag.Color = C.white
				d.distTag.Text = dist .. "s"
				d.distTag.Position = Vector2.new(x + w / 2, y + h + 12)
			else
				d.distTag.Visible = false
			end
		else
			d.distTag.Visible = false
		end

		if esp.showHealth then
			local hum = char:FindFirstChildOfClass("Humanoid")
			local hpRatio = hum and math.clamp(hum.Health / hum.MaxHealth, 0, 1) or 0
			local BAR_W = 3
			local bgX = x - BAR_W - 2

			d.healthBg.Visible = true
			d.healthBg.Position = Vector2.new(bgX, y)
			d.healthBg.Size = Vector2.new(BAR_W, h)
			d.healthBg.Color = Color3.fromRGB(0, 0, 0)

			local hpColor
			if hpRatio > 0.5 then
				hpColor = Color3.fromRGB(math.floor(255 * (1 - hpRatio) * 2), 200, 0)
			else
				hpColor = Color3.fromRGB(210, math.floor(200 * hpRatio * 2), 0)
			end

			d.healthFill.Visible = true
			d.healthFill.Color = hpColor
			d.healthFill.Position = Vector2.new(bgX, y + h * (1 - hpRatio))
			d.healthFill.Size = Vector2.new(BAR_W, h * hpRatio)
		else
			d.healthBg.Visible = false
			d.healthFill.Visible = false
		end
	end

	if not cfg.locked then return end

	if not target or not target.Character or not isAlive(target.Character) or not target.Character:FindFirstChild("HumanoidRootPart") then
		target = getClosest()
	end

	if not target then return end
	local char = target.Character
	if not char then return end
	local aim = aimPoint(char)
	if not aim then return end

	-- Corrected camera lock logic
	local currentCFrame = camera.CFrame
	local goalCFrame = CFrame.new(currentCFrame.Position, aim)

	-- Calculate the interpolation factor (alpha) based on lockSpeed and delta time
	-- The formula `1 - (1 - lockSpeed)^dt` provides a frame-rate independent smoothing.
	local alpha = 1 - math.pow(1 - cfg.lockSpeed, dt * 60) -- Multiply by 60 for consistent smoothing across different frame rates

	-- Apply the Lerp
	camera.CFrame = currentCFrame:Lerp(goalCFrame, alpha)

	-- Optional: Add a check for large angle differences to snap more quickly
	local angleDiff = math.acos(math.clamp(currentCFrame.LookVector:Dot(goalCFrame.LookVector), -1, 1))
	if angleDiff > math.rad(12) then
		camera.CFrame = currentCFrame:Lerp(goalCFrame, math.min(alpha * 1.5, 1)) -- Increase alpha for faster snapping
	end
end)

task.spawn(function()
	while true do
		task.wait(15)
		autoSave()
	end
end)# AIMBOT2
