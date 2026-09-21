--========================================================
-- KS HUB - MURDER MYSTERY
-- VISUAL POLISH + LOADING SCREEN + CREDITS
-- Mantém as funções: ESP / TRACERS / SPEED / SUPER JUMP
--========================================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer

--========================================================
-- CONFIG
--========================================================

local state = {
	ESP = true,
	Tracers = true,

	Speed = false,
	SpeedValue = 30,

	SuperJump = false,
	JumpPower = 100,
}

local ROLE_COLORS = {
	Murderer = Color3.fromRGB(255, 55, 55),
	Sheriff = Color3.fromRGB(70, 145, 255),
	Innocent = Color3.fromRGB(70, 255, 105),
	Unknown = Color3.fromRGB(255, 210, 70)
}

local MURDER_KEYWORDS = {
	"knife",
	"faca",
	"blade",
	"dagger",
	"sword",
	"murder"
}

local SHERIFF_KEYWORDS = {
	"gun",
	"pistol",
	"revolver",
	"weapon",
	"rifle",
	"sheriff"
}

local espObjects = {}

--========================================================
-- VISUAL HELPERS
--========================================================

local function tween(instance, info, props)
	local t = TweenService:Create(instance, info, props)
	t:Play()
	return t
end

local TI_FAST = TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local TI_SMOOTH = TweenInfo.new(0.28, Enum.EasingStyle.Quint, Enum.EasingDirection.Out)
local TI_SPRING = TweenInfo.new(0.42, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
local TI_FADE = TweenInfo.new(0.35, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)

local function addCorner(parent, radius)
	local c = Instance.new("UICorner")
	corner = c
	c.CornerRadius = UDim.new(0, radius or 10)
	c.Parent = parent
	return c
end

local function addStroke(parent, color, transparency, thickness)
	local s = Instance.new("UIStroke")
	s.Color = color or Color3.fromRGB(140, 80, 220)
	s.Transparency = transparency or 0.5
	s.Thickness = thickness or 1
	s.Parent = parent
	return s
end

local function addGradient(parent, color1, color2, rotation)
	local g = Instance.new("UIGradient")
	g.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, color1),
		ColorSequenceKeypoint.new(1, color2),
	})
	g.Rotation = rotation or 90
	g.Parent = parent
	return g
end

local function addShadow(parent)
	local shadow = Instance.new("ImageLabel")
	shadow.Name = "Shadow"
	shadow.BackgroundTransparency = 1
	shadow.Image = "rbxassetid://1316045217"
	shadow.ImageTransparency = 0.35
	shadow.ScaleType = Enum.ScaleType.Slice
	shadow.SliceCenter = Rect.new(10, 10, 118, 118)
	shadow.Size = UDim2.new(1, 30, 1, 30)
	shadow.Position = UDim2.fromOffset(-15, -15)
	shadow.ZIndex = math.max(parent.ZIndex - 1, 0)
	shadow.Parent = parent
	return shadow
end

local function styleButton(button, normalColor, hoverColor)
	button.AutoButtonColor = false
	addCorner(button, 10)
	addStroke(button, Color3.fromRGB(160, 105, 230), 0.78, 1)

	local baseSize = button.Size
	local basePosition = button.Position

	button.MouseEnter:Connect(function()
		tween(button, TI_FAST, {
			BackgroundColor3 = hoverColor,
			Size = UDim2.new(
				baseSize.X.Scale,
				baseSize.X.Offset + 2,
				baseSize.Y.Scale,
				baseSize.Y.Offset + 2
			),
			Position = UDim2.new(
				basePosition.X.Scale,
				basePosition.X.Offset - 1,
				basePosition.Y.Scale,
				basePosition.Y.Offset - 1
			)
		})
	end)

	button.MouseLeave:Connect(function()
		tween(button, TI_FAST, {
			BackgroundColor3 = normalColor,
			Size = baseSize,
			Position = basePosition
		})
	end)

	button.MouseButton1Down:Connect(function()
		tween(button, TI_FAST, {
			Size = UDim2.new(
				baseSize.X.Scale,
				baseSize.X.Offset - 2,
				baseSize.Y.Scale,
				baseSize.Y.Offset - 2
			)
		})
	end)

	button.MouseButton1Up:Connect(function()
		tween(button, TI_FAST, {
			Size = baseSize
		})
	end)
end

--========================================================
-- CHARACTER
--========================================================

local function getCharacter()
	local character = LocalPlayer.Character

	if not character then
		return nil, nil, nil
	end

	local humanoid = character:FindFirstChildOfClass("Humanoid")
	local root = character:FindFirstChild("HumanoidRootPart")

	return character, humanoid, root
end

--========================================================
-- ROLE DETECTION
--========================================================

local function getAllTools(player)
	local tools = {}

	local character = player.Character

	if character then
		for _, obj in ipairs(character:GetChildren()) do
			if obj:IsA("Tool") then
				table.insert(tools, obj)
			end
		end
	end

	local backpack = player:FindFirstChild("Backpack")

	if backpack then
		for _, obj in ipairs(backpack:GetChildren()) do
			if obj:IsA("Tool") then
				table.insert(tools, obj)
			end
		end
	end

	return tools
end

local function containsKeyword(name, keywords)
	name = string.lower(name)

	for _, keyword in ipairs(keywords) do
		if string.find(name, keyword, 1, true) then
			return true
		end
	end

	return false
end

local function getRole(player)
	local tools = getAllTools(player)

	if #tools == 0 then
		return "Innocent"
	end

	local murderTool = false
	local sheriffTool = false

	for _, tool in ipairs(tools) do
		if containsKeyword(tool.Name, MURDER_KEYWORDS) then
			murderTool = true
		end

		if containsKeyword(tool.Name, SHERIFF_KEYWORDS) then
			sheriffTool = true
		end
	end

	if sheriffTool then
		return "Sheriff"
	end

	if murderTool then
		return "Murderer"
	end

	return "Unknown"
end

--========================================================
-- ESP
--========================================================

local function removeESP(player)
	local data = espObjects[player]

	if not data then
		return
	end

	if data.Highlight then
		data.Highlight:Destroy()
	end

	if data.Billboard then
		data.Billboard:Destroy()
	end

	if data.TargetAttachment then
		data.TargetAttachment:Destroy()
	end

	if data.Beam then
		data.Beam:Destroy()
	end

	espObjects[player] = nil
end

local function getLocalAttachment()
	local _, _, root = getCharacter()

	if not root then
		return nil
	end

	local attachment = root:FindFirstChild("KS_TracerOrigin")

	if not attachment then
		attachment = Instance.new("Attachment")
		attachment.Name = "KS_TracerOrigin"
		attachment.Parent = root
	end

	return attachment
end

local function createESP(player)
	if player == LocalPlayer then
		return
	end

	local character = player.Character

	if not character then
		return
	end

	local root = character:FindFirstChild("HumanoidRootPart")

	if not root then
		return
	end

	removeESP(player)

	local role = getRole(player)
	local color = ROLE_COLORS[role] or ROLE_COLORS.Unknown

	local highlight = Instance.new("Highlight")
	highlight.Name = "KS_MurderESP"
	highlight.Adornee = character
	highlight.FillColor = color
	highlight.OutlineColor = color
	highlight.FillTransparency = 0.5
	highlight.OutlineTransparency = 0
	highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
	highlight.Parent = character

	local billboard = Instance.new("BillboardGui")
	billboard.Name = "KS_MurderESP_Name"
	billboard.Adornee = root
	billboard.Size = UDim2.fromOffset(200, 55)
	billboard.StudsOffset = Vector3.new(0, 3.2, 0)
	billboard.AlwaysOnTop = true
	billboard.Parent = root

	local text = Instance.new("TextLabel")
	text.Name = "RoleText"
	text.Size = UDim2.fromScale(1, 1)
	text.BackgroundTransparency = 1
	text.Text = player.DisplayName .. "\n[" .. role .. "]"
	text.TextColor3 = color
	text.TextStrokeTransparency = 0
	text.TextScaled = true
	text.Font = Enum.Font.GothamBold
	text.Parent = billboard

	local targetAttachment

	if state.Tracers then
		targetAttachment = Instance.new("Attachment")
		targetAttachment.Name = "KS_TracerTarget"
		targetAttachment.Parent = root
	end

	local beam

	if targetAttachment then
		local localAttachment = getLocalAttachment()

		if localAttachment then
			beam = Instance.new("Beam")
			beam.Name = "KS_PlayerTracer"
			beam.Attachment0 = localAttachment
			beam.Attachment1 = targetAttachment
			beam.Color = ColorSequence.new(color)
			beam.Width0 = 0.08
			beam.Width1 = 0.08
			beam.FaceCamera = true
			beam.LightEmission = 1
			beam.Transparency = NumberSequence.new(0.15)
			beam.Segments = 12
			beam.Parent = root
		end
	end

	espObjects[player] = {
		Character = character,
		Root = root,
		Role = role,
		Highlight = highlight,
		Billboard = billboard,
		Text = text,
		TargetAttachment = targetAttachment,
		Beam = beam
	}
end

local function updateESP(player)
	if player == LocalPlayer then
		return
	end

	if not state.ESP then
		removeESP(player)
		return
	end

	local character = player.Character

	if not character then
		removeESP(player)
		return
	end

	local root = character:FindFirstChild("HumanoidRootPart")

	if not root then
		removeESP(player)
		return
	end

	local data = espObjects[player]

	if not data or data.Character ~= character or data.Root ~= root then
		createESP(player)
		return
	end

	local role = getRole(player)
	local color = ROLE_COLORS[role] or ROLE_COLORS.Unknown

	data.Role = role

	if data.Highlight then
		data.Highlight.FillColor = color
		data.Highlight.OutlineColor = color
	end

	if data.Text then
		data.Text.Text = player.DisplayName .. "\n[" .. role .. "]"
		data.Text.TextColor3 = color
	end

	if state.Tracers and data.Beam then
		data.Beam.Enabled = true
		data.Beam.Color = ColorSequence.new(color)
	elseif data.Beam then
		data.Beam.Enabled = false
	end
end

local function refreshAll()
	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LocalPlayer then
			updateESP(player)
		end
	end
end

local function setupPlayer(player)
	if player == LocalPlayer then
		return
	end

	player.CharacterAdded:Connect(function()
		task.wait(0.4)
		updateESP(player)
	end)

	if player.Character then
		task.defer(function()
			updateESP(player)
		end)
	end
end

Players.PlayerAdded:Connect(setupPlayer)

Players.PlayerRemoving:Connect(function(player)
	removeESP(player)
end)

for _, player in ipairs(Players:GetPlayers()) do
	setupPlayer(player)
end

--========================================================
-- SPEED
--========================================================

local function applySpeed()
	local _, humanoid = getCharacter()

	if not humanoid then
		return
	end

	if state.Speed then
		humanoid.WalkSpeed = state.SpeedValue
	end
end

--========================================================
-- SUPER JUMP
--========================================================

local function applyJump()
	local _, humanoid = getCharacter()

	if not humanoid then
		return
	end

	if state.SuperJump then
		humanoid.UseJumpPower = true
		humanoid.JumpPower = state.JumpPower
	end
end

--========================================================
-- GUI
--========================================================

local playerGui = LocalPlayer:WaitForChild("PlayerGui")

local oldGui = playerGui:FindFirstChild("KS_MurderHub")
if oldGui then
	oldGui:Destroy()
end

local gui = Instance.new("ScreenGui")
gui.Name = "KS_MurderHub"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.Parent = playerGui

--========================================================
-- BLUR / LOADING
--========================================================

local blur = Instance.new("BlurEffect")
blur.Name = "KS_MurderHub_Blur"
blur.Size = 0
blur.Parent = Lighting

local loading = Instance.new("Frame")
loading.Name = "Loading"
loading.Size = UDim2.fromScale(1, 1)
loading.BackgroundColor3 = Color3.fromRGB(8, 6, 14)
loading.BorderSizePixel = 0
loading.ZIndex = 100
loading.Parent = gui

local loadingGradient = addGradient(
	loading,
	Color3.fromRGB(16, 9, 27),
	Color3.fromRGB(7, 6, 12),
	135
)

local loadingTitle = Instance.new("TextLabel")
loadingTitle.AnchorPoint = Vector2.new(0.5, 0.5)
loadingTitle.Position = UDim2.fromScale(0.5, 0.39)
loadingTitle.Size = UDim2.fromOffset(700, 70)
loadingTitle.BackgroundTransparency = 1
loadingTitle.Text = "KS HUB"
loadingTitle.TextColor3 = Color3.new(1, 1, 1)
loadingTitle.TextTransparency = 1
loadingTitle.TextSize = 48
loadingTitle.Font = Enum.Font.GothamBlack
loadingTitle.ZIndex = 101
loadingTitle.Parent = loading

local loadingSub = Instance.new("TextLabel")
loadingSub.AnchorPoint = Vector2.new(0.5, 0.5)
loadingSub.Position = UDim2.fromScale(0.5, 0.47)
loadingSub.Size = UDim2.fromOffset(700, 45)
loadingSub.BackgroundTransparency = 1
loadingSub.Text = "MURDER MYSTERY"
loadingSub.TextColor3 = Color3.fromRGB(181, 135, 245)
loadingSub.TextTransparency = 1
loadingSub.TextSize = 23
loadingSub.Font = Enum.Font.GothamBold
loadingSub.ZIndex = 101
loadingSub.Parent = loading

local line = Instance.new("Frame")
line.AnchorPoint = Vector2.new(0.5, 0.5)
line.Position = UDim2.fromScale(0.5, 0.55)
line.Size = UDim2.fromOffset(0, 2)
line.BackgroundColor3 = Color3.fromRGB(138, 74, 220)
line.BorderSizePixel = 0
line.ZIndex = 101
line.Parent = loading

local lineGlow = addStroke(line, Color3.fromRGB(185, 130, 255), 0.35, 1)

local loadingStatus = Instance.new("TextLabel")
loadingStatus.AnchorPoint = Vector2.new(0.5, 0.5)
loadingStatus.Position = UDim2.fromScale(0.5, 0.61)
loadingStatus.Size = UDim2.fromOffset(400, 28)
loadingStatus.BackgroundTransparency = 1
loadingStatus.Text = "INITIALIZING..."
loadingStatus.TextColor3 = Color3.fromRGB(185, 185, 195)
loadingStatus.TextTransparency = 1
loadingStatus.TextSize = 13
loadingStatus.Font = Enum.Font.GothamMedium
loadingStatus.ZIndex = 101
loadingStatus.Parent = loading

local loadingDiscord = Instance.new("TextLabel")
loadingDiscord.AnchorPoint = Vector2.new(0.5, 0.5)
loadingDiscord.Position = UDim2.fromScale(0.5, 0.66)
loadingDiscord.Size = UDim2.fromOffset(500, 26)
loadingDiscord.BackgroundTransparency = 1
loadingDiscord.Text = "discord.gg/7Rkcbztcy8"
loadingDiscord.TextColor3 = Color3.fromRGB(145, 112, 200)
loadingDiscord.TextTransparency = 1
loadingDiscord.TextSize = 12
loadingDiscord.Font = Enum.Font.GothamMedium
loadingDiscord.ZIndex = 101
loadingDiscord.Parent = loading

local progressBack = Instance.new("Frame")
progressBack.AnchorPoint = Vector2.new(0.5, 0.5)
progressBack.Position = UDim2.fromScale(0.5, 0.72)
progressBack.Size = UDim2.fromOffset(360, 5)
progressBack.BackgroundColor3 = Color3.fromRGB(37, 28, 49)
progressBack.BorderSizePixel = 0
progressBack.ZIndex = 101
progressBack.Parent = loading
addCorner(progressBack, 10)

local progress = Instance.new("Frame")
progress.Size = UDim2.new(0, 0, 1, 0)
progress.BackgroundColor3 = Color3.fromRGB(152, 86, 233)
progress.BorderSizePixel = 0
progress.ZIndex = 102
progress.Parent = progressBack
addCorner(progress, 10)
addGradient(progress, Color3.fromRGB(110, 60, 190), Color3.fromRGB(196, 120, 255), 0)

local complete = Instance.new("TextLabel")
complete.AnchorPoint = Vector2.new(0.5, 0.5)
complete.Position = UDim2.fromScale(0.5, 0.77)
complete.Size = UDim2.fromOffset(500, 30)
complete.BackgroundTransparency = 1
complete.Text = "COMPLETED"
complete.TextColor3 = Color3.fromRGB(125, 255, 180)
complete.TextTransparency = 1
complete.TextSize = 13
complete.Font = Enum.Font.GothamBold
complete.ZIndex = 101
complete.Parent = loading

--========================================================
-- MAIN PANEL - BLACK / WHITE PROFESSIONAL STYLE
--========================================================

local main = Instance.new("Frame")
main.Name = "Main"
main.Size = UDim2.fromOffset(530, 425)
main.Position = UDim2.new(0.5, -265, 0.5, -212)
main.BackgroundColor3 = Color3.fromRGB(10, 6, 16)
main.BackgroundTransparency = 1
main.BorderSizePixel = 0
main.Visible = false
main.ZIndex = 10
main.Parent = gui

addCorner(main, 16)

-- White premium outline
local outerStroke = addStroke(main, Color3.fromRGB(167, 73, 255), 0.06, 2)

-- Soft shadow, kept subtle so the panel stays black
local shadow = addShadow(main)
shadow.ImageTransparency = 0.42

-- Small purple accent line at the very top
local accent = Instance.new("Frame")
accent.Position = UDim2.fromOffset(18, 0)
accent.Size = UDim2.new(1, -36, 0, 2)
accent.BackgroundColor3 = Color3.fromRGB(172, 65, 255)
accent.BorderSizePixel = 0
accent.ZIndex = 20
accent.Parent = main
addCorner(accent, 4)

--========================================================
-- TOP BAR
--========================================================

local top = Instance.new("Frame")
top.Size = UDim2.new(1, 0, 0, 66)
top.BackgroundColor3 = Color3.fromRGB(78, 24, 145)
top.BorderSizePixel = 0
top.ZIndex = 11
top.Parent = main
addCorner(top, 16)

-- Covers only the lower rounded part of the top bar
local topMask = Instance.new("Frame")
topMask.Position = UDim2.fromOffset(0, 47)
topMask.Size = UDim2.new(1, 0, 0, 19)
topMask.BackgroundColor3 = Color3.fromRGB(66, 20, 122)
topMask.BorderSizePixel = 0
topMask.ZIndex = 11
topMask.Parent = top

local topDivider = Instance.new("Frame")
topDivider.Position = UDim2.fromOffset(18, 65)
topDivider.Size = UDim2.new(1, -36, 0, 1)
topDivider.BackgroundColor3 = Color3.fromRGB(214, 176, 255)
topDivider.BackgroundTransparency = 0.68
topDivider.BorderSizePixel = 0
topDivider.ZIndex = 12
topDivider.Parent = top

local title = Instance.new("TextLabel")
title.Position = UDim2.fromOffset(22, 7)
title.Size = UDim2.fromOffset(400, 31)
title.BackgroundTransparency = 1
title.Text = "KS HUB"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextSize = 22
title.Font = Enum.Font.GothamBlack
title.TextXAlignment = Enum.TextXAlignment.Left
title.ZIndex = 12
title.Parent = top

local subtitle = Instance.new("TextLabel")
subtitle.Position = UDim2.fromOffset(23, 37)
subtitle.Size = UDim2.fromOffset(400, 18)
subtitle.BackgroundTransparency = 1
subtitle.Text = "MURDER MYSTERY"
subtitle.TextColor3 = Color3.fromRGB(164, 124, 218)
subtitle.TextSize = 10
subtitle.Font = Enum.Font.GothamBold
subtitle.TextXAlignment = Enum.TextXAlignment.Left
subtitle.ZIndex = 12
subtitle.Parent = top

local statusDot = Instance.new("Frame")
statusDot.Position = UDim2.fromOffset(157, 43)
statusDot.Size = UDim2.fromOffset(6, 6)
statusDot.BackgroundColor3 = Color3.fromRGB(105, 255, 161)
statusDot.BorderSizePixel = 0
statusDot.ZIndex = 12
statusDot.Parent = top
addCorner(statusDot, 8)

local creditsButton = Instance.new("TextButton")
creditsButton.AnchorPoint = Vector2.new(1, 0.5)
creditsButton.Position = UDim2.new(1, -16, 0.5, 0)
creditsButton.Size = UDim2.fromOffset(96, 34)
creditsButton.BackgroundColor3 = Color3.fromRGB(94, 34, 157)
creditsButton.Text = "CREDITS"
creditsButton.TextColor3 = Color3.fromRGB(235, 229, 242)
creditsButton.TextSize = 10
creditsButton.Font = Enum.Font.GothamBold
creditsButton.BorderSizePixel = 0
creditsButton.ZIndex = 12
creditsButton.Parent = top
styleButton(
	creditsButton,
	Color3.fromRGB(94, 34, 157),
	Color3.fromRGB(126, 49, 205)
)

--========================================================
-- BODY / CARDS
--========================================================

local function createCard(y, h)
	local card = Instance.new("Frame")
	card.Position = UDim2.fromOffset(18, y)
	card.Size = UDim2.new(1, -36, 0, h)
	card.BackgroundColor3 = Color3.fromRGB(15, 9, 23)
	card.BorderSizePixel = 0
	card.ZIndex = 11
	card.Parent = main
	addCorner(card, 12)
	addStroke(card, Color3.fromRGB(171, 87, 255), 0.52, 1)
	return card
end

local visualCard = createCard(82, 112)
local movementCard = createCard(206, 140)

local function createSectionTitle(card, textValue)
	local label = Instance.new("TextLabel")
	label.Position = UDim2.fromOffset(14, 10)
	label.Size = UDim2.new(1, -28, 0, 22)
	label.BackgroundTransparency = 1
	label.Text = textValue
	label.TextColor3 = Color3.fromRGB(220, 220, 224)
	label.TextSize = 11
	label.Font = Enum.Font.GothamBold
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.ZIndex = 12
	label.Parent = card
	return label
end

createSectionTitle(visualCard, "VISUAL")
createSectionTitle(movementCard, "MOVEMENT")

local function createToggle(card, x, y, width, textValue, default, callback)
	local button = Instance.new("TextButton")
	button.Position = UDim2.fromOffset(x, y)
	button.Size = UDim2.fromOffset(width, 38)
	button.BackgroundColor3 = default and Color3.fromRGB(108, 39, 177) or Color3.fromRGB(30, 25, 36)
	button.BorderSizePixel = 0
	button.TextColor3 = Color3.fromRGB(247, 247, 249)
	button.TextSize = 11
	button.Font = Enum.Font.GothamBold
	button.AutoButtonColor = false
	button.ZIndex = 12
	button.Parent = card

	addCorner(button, 9)
	local buttonStroke = addStroke(
		button,
		default and Color3.fromRGB(203, 119, 255) or Color3.fromRGB(255, 255, 255),
		default and 0.30 or 0.84,
		1
	)

	local dot = Instance.new("Frame")
	dot.AnchorPoint = Vector2.new(1, 0.5)
	dot.Position = UDim2.new(1, -10, 0.5, 0)
	dot.Size = UDim2.fromOffset(7, 7)
	dot.BackgroundColor3 = default and Color3.fromRGB(100, 255, 165) or Color3.fromRGB(140, 140, 148)
	dot.BorderSizePixel = 0
	dot.ZIndex = 13
	dot.Parent = button
	addCorner(dot, 10)

	local value = default

	local function refresh(newValue)
		value = newValue
		button.Text = textValue .. "     " .. (newValue and "ON" or "OFF")

		tween(button, TI_FAST, {
			BackgroundColor3 = newValue and Color3.fromRGB(108, 39, 177) or Color3.fromRGB(30, 25, 36)
		})

		tween(dot, TI_FAST, {
			BackgroundColor3 = newValue and Color3.fromRGB(100, 255, 165) or Color3.fromRGB(140, 140, 148)
		})

		buttonStroke.Color = newValue and Color3.fromRGB(203, 119, 255) or Color3.fromRGB(255, 255, 255)
		buttonStroke.Transparency = newValue and 0.30 or 0.84
	end

	button.MouseEnter:Connect(function()
		tween(button, TI_FAST, {
			BackgroundColor3 = value and Color3.fromRGB(130, 48, 211) or Color3.fromRGB(43, 35, 51)
		})
	end)

	button.MouseLeave:Connect(function()
		refresh(value)
	end)

	button.MouseButton1Click:Connect(function()
		value = not value
		refresh(value)
		callback(value)
	end)

	refresh(value)
	return button
end

--========================================================
-- VISUAL BUTTONS
--========================================================

createToggle(visualCard, 14, 42, 210, "PLAYER ESP", state.ESP, function(value)
	state.ESP = value
	refreshAll()
end)

createToggle(visualCard, 238, 42, 210, "TRACERS", state.Tracers, function(value)
	state.Tracers = value

	for player in pairs(espObjects) do
		removeESP(player)
	end

	refreshAll()
end)

local legend = Instance.new("TextLabel")
legend.Position = UDim2.fromOffset(14, 86)
legend.Size = UDim2.new(1, -28, 0, 18)
legend.BackgroundTransparency = 1
legend.Text = "● Murderer     ● Sheriff     ● Innocent"
legend.TextColor3 = Color3.fromRGB(183, 183, 188)
legend.TextSize = 10
legend.Font = Enum.Font.GothamMedium
legend.TextXAlignment = Enum.TextXAlignment.Left
legend.ZIndex = 12
legend.Parent = visualCard

-- Color the role names individually without changing the existing ESP logic.
local legendM = Instance.new("TextLabel")
legendM.Position = UDim2.fromOffset(14, 86)
legendM.Size = UDim2.fromOffset(86, 18)
legendM.BackgroundTransparency = 1
legendM.Text = "● Murderer"
legendM.TextColor3 = ROLE_COLORS.Murderer
legendM.TextSize = 10
legendM.Font = Enum.Font.GothamMedium
legendM.TextXAlignment = Enum.TextXAlignment.Left
legendM.ZIndex = 13
legendM.Parent = visualCard

local legendS = Instance.new("TextLabel")
legendS.Position = UDim2.fromOffset(101, 86)
legendS.Size = UDim2.fromOffset(80, 18)
legendS.BackgroundTransparency = 1
legendS.Text = "● Sheriff"
legendS.TextColor3 = ROLE_COLORS.Sheriff
legendS.TextSize = 10
legendS.Font = Enum.Font.GothamMedium
legendS.TextXAlignment = Enum.TextXAlignment.Left
legendS.ZIndex = 13
legendS.Parent = visualCard

local legendI = Instance.new("TextLabel")
legendI.Position = UDim2.fromOffset(183, 86)
legendI.Size = UDim2.fromOffset(100, 18)
legendI.BackgroundTransparency = 1
legendI.Text = "● Innocent"
legendI.TextColor3 = ROLE_COLORS.Innocent
legendI.TextSize = 10
legendI.Font = Enum.Font.GothamMedium
legendI.TextXAlignment = Enum.TextXAlignment.Left
legendI.ZIndex = 13
legendI.Parent = visualCard

legend.Visible = false

--========================================================
-- NUMBER INPUT
--========================================================

local function createNumberBox(card, x, y, labelText, defaultValue)
	local label = Instance.new("TextLabel")
	label.Position = UDim2.fromOffset(x, y + 1)
	label.Size = UDim2.fromOffset(48, 34)
	label.BackgroundTransparency = 1
	label.Text = labelText
	label.TextColor3 = Color3.fromRGB(178, 178, 185)
	label.TextSize = 10
	label.Font = Enum.Font.GothamBold
	label.TextXAlignment = Enum.TextXAlignment.Right
	label.ZIndex = 12
	label.Parent = card

	local box = Instance.new("TextBox")
	box.Position = UDim2.fromOffset(x + 58, y)
	box.Size = UDim2.fromOffset(82, 36)
	box.BackgroundColor3 = Color3.fromRGB(22, 14, 31)
	box.BorderSizePixel = 0
	box.Text = tostring(defaultValue)
	box.TextColor3 = Color3.fromRGB(248, 248, 250)
	box.TextSize = 12
	box.Font = Enum.Font.GothamBold
	box.ClearTextOnFocus = false
	box.ZIndex = 12
	box.Parent = card

	addCorner(box, 9)
	local boxStroke = addStroke(box, Color3.fromRGB(194, 113, 255), 0.62, 1)

	box.Focused:Connect(function()
		tween(box, TI_FAST, {
			BackgroundColor3 = Color3.fromRGB(37, 22, 51)
		})
		boxStroke.Transparency = 0.26
	end)

	box.FocusLost:Connect(function()
		tween(box, TI_FAST, {
			BackgroundColor3 = Color3.fromRGB(22, 14, 31)
		})
		boxStroke.Transparency = 0.62
	end)

	return box
end

--========================================================
-- MOVEMENT CONTROLS
--========================================================

createToggle(movementCard, 14, 42, 210, "SPEED", state.Speed, function(value)
	state.Speed = value

	if state.Speed then
		applySpeed()
	else
		local _, humanoid = getCharacter()
		if humanoid then
			humanoid.WalkSpeed = 16
		end
	end
end)

local speedBox = createNumberBox(movementCard, 276, 43, "Value", state.SpeedValue)

speedBox.FocusLost:Connect(function()
	local number = tonumber(speedBox.Text)

	if number then
		state.SpeedValue = math.clamp(number, 0, 250)
		speedBox.Text = tostring(state.SpeedValue)

		if state.Speed then
			applySpeed()
		end
	else
		speedBox.Text = tostring(state.SpeedValue)
	end
end)

createToggle(movementCard, 14, 92, 210, "SUPER JUMP", state.SuperJump, function(value)
	state.SuperJump = value

	if state.SuperJump then
		applyJump()
	else
		local _, humanoid = getCharacter()
		if humanoid then
			humanoid.UseJumpPower = true
			humanoid.JumpPower = 50
		end
	end
end)

local jumpBox = createNumberBox(movementCard, 276, 93, "Power", state.JumpPower)

jumpBox.FocusLost:Connect(function()
	local number = tonumber(jumpBox.Text)

	if number then
		state.JumpPower = math.clamp(number, 50, 300)
		jumpBox.Text = tostring(state.JumpPower)

		if state.SuperJump then
			applyJump()
		end
	else
		jumpBox.Text = tostring(state.JumpPower)
	end
end)

--========================================================
-- CREDITS MODAL
--========================================================

local overlay = Instance.new("Frame")
overlay.Name = "CreditsOverlay"
overlay.Size = UDim2.fromScale(1, 1)
overlay.BackgroundColor3 = Color3.fromRGB(7, 3, 15)
overlay.BackgroundTransparency = 1
overlay.Visible = false
overlay.ZIndex = 50
overlay.Parent = gui

local creditsCard = Instance.new("Frame")
creditsCard.AnchorPoint = Vector2.new(0.5, 0.5)
creditsCard.Position = UDim2.fromScale(0.5, 0.53)
creditsCard.Size = UDim2.fromOffset(390, 250)
creditsCard.BackgroundColor3 = Color3.fromRGB(22, 10, 36)
creditsCard.BorderSizePixel = 0
creditsCard.ZIndex = 51
creditsCard.Parent = overlay

addCorner(creditsCard, 16)
addStroke(creditsCard, Color3.fromRGB(197, 93, 255), 0.15, 2)
addShadow(creditsCard)
addGradient(
	creditsCard,
	Color3.fromRGB(53, 18, 86),
	Color3.fromRGB(17, 7, 27),
	135
)

local creditsTitle = Instance.new("TextLabel")
creditsTitle.Position = UDim2.fromOffset(22, 20)
creditsTitle.Size = UDim2.new(1, -44, 0, 28)
creditsTitle.BackgroundTransparency = 1
creditsTitle.Text = "KS HUB"
creditsTitle.TextColor3 = Color3.new(1, 1, 1)
creditsTitle.TextSize = 23
creditsTitle.Font = Enum.Font.GothamBlack
creditsTitle.TextXAlignment = Enum.TextXAlignment.Left
creditsTitle.ZIndex = 52
creditsTitle.Parent = creditsCard

local creditsSub = Instance.new("TextLabel")
creditsSub.Position = UDim2.fromOffset(22, 49)
creditsSub.Size = UDim2.new(1, -44, 0, 20)
creditsSub.BackgroundTransparency = 1
creditsSub.Text = "Created by Kalebzin Studio"
creditsSub.TextColor3 = Color3.fromRGB(224, 187, 255)
creditsSub.TextSize = 12
creditsSub.Font = Enum.Font.GothamMedium
creditsSub.TextXAlignment = Enum.TextXAlignment.Left
creditsSub.ZIndex = 52
creditsSub.Parent = creditsCard

local divider = Instance.new("Frame")
divider.Position = UDim2.fromOffset(22, 82)
divider.Size = UDim2.new(1, -44, 0, 1)
divider.BackgroundColor3 = Color3.fromRGB(139, 64, 198)
divider.BorderSizePixel = 0
divider.ZIndex = 52
divider.Parent = creditsCard

local discordLabel = Instance.new("TextLabel")
discordLabel.Position = UDim2.fromOffset(22, 100)
discordLabel.Size = UDim2.new(1, -44, 0, 20)
discordLabel.BackgroundTransparency = 1
discordLabel.Text = "DISCORD"
discordLabel.TextColor3 = Color3.fromRGB(210, 166, 246)
discordLabel.TextSize = 11
discordLabel.Font = Enum.Font.GothamBold
discordLabel.TextXAlignment = Enum.TextXAlignment.Left
discordLabel.ZIndex = 52
discordLabel.Parent = creditsCard

local discordBox = Instance.new("TextBox")
discordBox.Position = UDim2.fromOffset(22, 127)
discordBox.Size = UDim2.new(1, -44, 0, 40)
discordBox.BackgroundColor3 = Color3.fromRGB(31, 12, 47)
discordBox.BorderSizePixel = 0
discordBox.Text = "discord.gg/7Rkcbztcy8"
discordBox.TextColor3 = Color3.fromRGB(255, 255, 255)
discordBox.TextSize = 12
discordBox.Font = Enum.Font.GothamBold
discordBox.ClearTextOnFocus = false
discordBox.TextEditable = false
discordBox.ZIndex = 52
discordBox.Parent = creditsCard
addCorner(discordBox, 9)
addStroke(discordBox, Color3.fromRGB(173, 83, 239), 0.32, 1)

local copyButton = Instance.new("TextButton")
copyButton.Position = UDim2.fromOffset(22, 178)
copyButton.Size = UDim2.fromOffset(160, 42)
copyButton.BackgroundColor3 = Color3.fromRGB(116, 39, 191)
copyButton.BorderSizePixel = 0
copyButton.Text = "COPY DISCORD"
copyButton.TextColor3 = Color3.new(1, 1, 1)
copyButton.TextSize = 12
copyButton.Font = Enum.Font.GothamBold
copyButton.ZIndex = 52
copyButton.Parent = creditsCard
styleButton(
	copyButton,
	Color3.fromRGB(116, 39, 191),
	Color3.fromRGB(154, 59, 238)
)

local closeButton = Instance.new("TextButton")
closeButton.Position = UDim2.fromOffset(192, 178)
closeButton.Size = UDim2.fromOffset(160, 42)
closeButton.BackgroundColor3 = Color3.fromRGB(42, 20, 55)
closeButton.BorderSizePixel = 0
closeButton.Text = "CLOSE"
closeButton.TextColor3 = Color3.fromRGB(235, 230, 240)
closeButton.TextSize = 12
closeButton.Font = Enum.Font.GothamBold
closeButton.ZIndex = 52
closeButton.Parent = creditsCard
styleButton(
	closeButton,
	Color3.fromRGB(42, 20, 55),
	Color3.fromRGB(67, 31, 84)
)

local copySupported = typeof(setclipboard) == "function"

copyButton.MouseButton1Click:Connect(function()
	if copySupported then
		pcall(setclipboard, "https://discord.gg/7Rkcbztcy8")
	end

	local original = copyButton.Text
	copyButton.Text = copySupported and "COPIED ✓" or "SELECT DISCORD"

	task.delay(1.35, function()
		if copyButton and copyButton.Parent then
			copyButton.Text = original
		end
	end)
end)

local function openCredits()
	overlay.Visible = true
	creditsCard.Size = UDim2.fromOffset(360, 225)
	creditsCard.Position = UDim2.fromScale(0.5, 0.55)

	tween(overlay, TI_FADE, {
		BackgroundTransparency = 0.12
	})

	tween(creditsCard, TI_SPRING, {
		Size = UDim2.fromOffset(390, 250),
		Position = UDim2.fromScale(0.5, 0.5)
	})
end

local function closeCredits()
	tween(overlay, TI_FADE, {
		BackgroundTransparency = 1
	})

	tween(creditsCard, TI_FAST, {
		Size = UDim2.fromOffset(360, 225),
		Position = UDim2.fromScale(0.5, 0.55)
	})

	task.delay(0.18, function()
		if overlay and overlay.Parent then
			overlay.Visible = false
		end
	end)
end

creditsButton.MouseButton1Click:Connect(openCredits)
closeButton.MouseButton1Click:Connect(closeCredits)

--========================================================
-- RGB PURPLE GLOW
--========================================================

local rgbStrokes = { outerStroke }
local rgbAccents = { accent }

-- Add the credits outline to the same RGB cycle.
local creditsStroke = creditsCard:FindFirstChildOfClass("UIStroke")
if creditsStroke then
	table.insert(rgbStrokes, creditsStroke)
end

local rgbClock = 0
RunService.RenderStepped:Connect(function(dt)
	rgbClock += dt * 0.55

	local hue = (0.76 + math.sin(rgbClock) * 0.09) % 1
	local hue2 = (hue + 0.10) % 1

	local c1 = Color3.fromHSV(hue, 0.78, 1)
	local c2 = Color3.fromHSV(hue2, 0.78, 1)

	for _, stroke in ipairs(rgbStrokes) do
		if stroke and stroke.Parent then
			stroke.Color = c1
			stroke.Transparency = 0.12 + (math.sin(rgbClock * 1.7) + 1) * 0.08
		end
	end

	for _, obj in ipairs(rgbAccents) do
		if obj and obj.Parent then
			obj.BackgroundColor3 = c2
		end
	end
end)

--========================================================
-- DRAG
--========================================================

local dragging = false
local dragStart
local startPosition

top.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragStart = input.Position
		startPosition = main.Position
	end
end)

top.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = false
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
		local delta = input.Position - dragStart

		main.Position = UDim2.new(
			startPosition.X.Scale,
			startPosition.X.Offset + delta.X,
			startPosition.Y.Scale,
			startPosition.Y.Offset + delta.Y
		)
	end
end)

--========================================================
-- RESPawn
--========================================================

LocalPlayer.CharacterAdded:Connect(function()
	task.wait(0.7)

	applySpeed()
	applyJump()

	if state.ESP then
		task.defer(refreshAll)
	end
end)

--========================================================
-- LOADING ANIMATION
--========================================================

task.spawn(function()
	tween(blur, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Size = 16
	})

	tween(loadingTitle, TI_SMOOTH, {
		TextTransparency = 0
	})

	task.wait(0.12)

	tween(loadingSub, TI_SMOOTH, {
		TextTransparency = 0
	})

	task.wait(0.1)

	tween(loadingStatus, TI_SMOOTH, {
		TextTransparency = 0
	})

	tween(loadingDiscord, TI_SMOOTH, {
		TextTransparency = 0
	})

	tween(line, TweenInfo.new(0.65, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
		Size = UDim2.fromOffset(300, 2)
	})

	local statuses = {
		"INITIALIZING...",
		"LOADING PLAYER ESP...",
		"LOADING TRACERS...",
		"LOADING MOVEMENT...",
		"FINALIZING..."
	}

	for i, status in ipairs(statuses) do
		loadingStatus.Text = status

		tween(progress, TweenInfo.new(0.65, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
			Size = UDim2.new(i / #statuses, 0, 1, 0)
		})

		task.wait(0.65)
	end

	loadingStatus.Text = "READY"

	tween(complete, TI_SMOOTH, {
		TextTransparency = 0
	})

	task.wait(0.55)

	main.Visible = true
	main.Position = UDim2.new(0.5, -265, 0.52, -212)

	tween(main, TI_SPRING, {
		Position = UDim2.new(0.5, -265, 0.5, -212),
		BackgroundTransparency = 0
	})

	task.wait(0.18)

	tween(loading, TI_FADE, {
		BackgroundTransparency = 1
	})

	tween(blur, TweenInfo.new(0.45, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Size = 0
	})

	task.wait(0.45)

	loading.Visible = false

	if blur and blur.Parent then
		blur:Destroy()
	end
end)

--========================================================
-- UPDATE LOOP
--========================================================

task.spawn(function()
	while gui.Parent do
		task.wait(0.3)

		if state.ESP then
			refreshAll()
		end

		if state.Speed then
			applySpeed()
		end

		if state.SuperJump then
			applyJump()
		end
	end
end)

print("[KS HUB] Murder Mystery carregado.")
