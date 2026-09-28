--// ============================================================
--// MÓDULO ESP v1 — UNIVERSAL
--// PluginId: Batata012
--// IconId: nil (sem ícone)
--// Funciona em qualquer jogo
--// ============================================================

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local Cam = workspace.CurrentCamera

local api = ReplicatedStorage:WaitForChild("BatataHub_RegisterTab")

--// SAVE/LOAD
local CONFIG_FOLDER = "Batata Central"
local CONFIG_FILE = "esp.json"

local function getConfigPath()
	if not isfolder(CONFIG_FOLDER) then
		makefolder(CONFIG_FOLDER)
	end
	return CONFIG_FOLDER .. "/" .. CONFIG_FILE
end

local function saveESP(data)
	pcall(function()
		writefile(getConfigPath(), HttpService:JSONEncode(data))
	end)
end

local function loadESP()
	local path = getConfigPath()
	if isfile(path) then
		local ok, decoded = pcall(function()
			return HttpService:JSONDecode(readfile(path))
		end)
		if ok and type(decoded) == "table" then
			return decoded
		end
	end
	local default = {}
	saveESP(default)
	return default
end

local S = loadESP()

--// ESTADO (tudo desativado por padrão)
local ESP = {
	Enabled = S.Enabled == true or false,

	-- Filtros
	TeamCheck = S.TeamCheck == true or false,
	ShowFriends = S.ShowFriends == true or false,
	OnlyFriends = S.OnlyFriends == true or false,
	MaxDistance = S.MaxDistance or 1000,
	HideDead = S.HideDead ~= false,

	-- Visual
	Transparency = S.Transparency or 40,
	OutlineEnabled = S.OutlineEnabled ~= false,
	DepthMode = S.DepthMode or "AlwaysOnTop",

	-- Cores
	ColorEnemy = S.ColorEnemy or "Vermelho",
	ColorTeam = S.ColorTeam or "Azul",
	ColorFriend = S.ColorFriend or "Verde",

	-- Info
	ShowDistance = S.ShowDistance == true or false,
	ShowName = S.ShowName == true or false,
	ShowHealth = S.ShowHealth == true or false,
	ShowTarget = S.ShowTarget == true or false,

	-- Performance
	RefreshRate = S.RefreshRate or 0.1,
}

local Cores = {
	["Branco"]   = Color3.fromRGB(255, 255, 255),
	["Preto"]    = Color3.fromRGB(20, 20, 20),
	["Vermelho"] = Color3.fromRGB(255, 60, 60),
	["Verde"]    = Color3.fromRGB(60, 255, 100),
	["Azul"]     = Color3.fromRGB(80, 160, 255),
	["Amarelo"]  = Color3.fromRGB(255, 220, 80),
	["Roxo"]     = Color3.fromRGB(200, 100, 255),
	["Rosa"]     = Color3.fromRGB(255, 120, 200),
	["Ciano"]    = Color3.fromRGB(80, 240, 240),
	["Laranja"]  = Color3.fromRGB(255, 160, 60),
}

local function persist()
	saveESP({
		Enabled = ESP.Enabled,
		TeamCheck = ESP.TeamCheck,
		ShowFriends = ESP.ShowFriends,
		OnlyFriends = ESP.OnlyFriends,
		MaxDistance = ESP.MaxDistance,
		HideDead = ESP.HideDead,
		Transparency = ESP.Transparency,
		OutlineEnabled = ESP.OutlineEnabled,
		DepthMode = ESP.DepthMode,
		ColorEnemy = ESP.ColorEnemy,
		ColorTeam = ESP.ColorTeam,
		ColorFriend = ESP.ColorFriend,
		ShowDistance = ESP.ShowDistance,
		ShowName = ESP.ShowName,
		ShowHealth = ESP.ShowHealth,
		ShowTarget = ESP.ShowTarget,
		RefreshRate = ESP.RefreshRate,
	})
end
--// ============================================================
--// ESP — Amigos, Times e Filtros
--// ============================================================

--// Cache de amigos
local friendSet = {}

task.spawn(function()
	local ok, list = pcall(function()
		return Players:GetFriendsAsync(player.UserId)
	end)
	if ok and list then
		while true do
			for _, info in ipairs(list:GetCurrentPage()) do
				friendSet[info.Id] = true
			end
			if list.IsFinished then break end
			pcall(function() list:AdvanceToNextPageAsync() end)
		end
	end
	print("[ESP] Amigos carregados.")
end)

--// Checa se é amigo
local function isFriend(p)
	return friendSet[p.UserId] == true
end

--// Checa se é do mesmo time
local function isTeammate(p)
	if not p.Team or not player.Team then return false end
	return p.Team == player.Team
end

--// Checa se é o alvo do AIM (se existir)
local function isAimTarget(p)
	if not _G.Aim or not _G.Aim.CurrentTarget then return false end
	local ok, target = pcall(function() return _G.Aim.CurrentTarget end)
	return ok and target == p
end

--// Escolhe a cor
local function getESPColor(p)
	if isFriend(p) then
		return Cores[ESP.ColorFriend] or Color3.fromRGB(60, 255, 100)
	end
	if ESP.TeamCheck and isTeammate(p) then
		return Cores[ESP.ColorTeam] or Color3.fromRGB(80, 160, 255)
	end
	return Cores[ESP.ColorEnemy] or Color3.fromRGB(255, 60, 60)
end

--// Decide se um jogador deve ter ESP
local function shouldShowESP(p)
	if p == player then return false end
	if not p.Character then return false end

	-- Humanoid vivo?
	if ESP.HideDead then
		local hum = p.Character:FindFirstChildOfClass("Humanoid")
		if not hum or hum.Health <= 0 then return false end
	end

	-- Só amigos
	if ESP.OnlyFriends and not isFriend(p) then
		return false
	end

	-- Esconder amigos
	if not ESP.ShowFriends and isFriend(p) then
		return false
	end

	-- Team Check (esconde aliados se ativado e ShowFriends off)
	if ESP.TeamCheck and isTeammate(p) and not ESP.ShowFriends then
		return false
	end

	-- Distância máxima
	if ESP.MaxDistance > 0 then
		local myChar = player.Character
		if myChar and myChar:FindFirstChild("HumanoidRootPart") then
			local targetRoot = p.Character:FindFirstChild("HumanoidRootPart")
			if targetRoot then
				local dist = (myChar.HumanoidRootPart.Position - targetRoot.Position).Magnitude
				if dist > ESP.MaxDistance then
					return false
				end
			end
		end
	end

	return true
end
--// ============================================================
--// ESP — Sistema de Highlight
//// ============================================================

local espFolder = Instance.new("Folder")
espFolder.Name = "BatataHub_ESP"
espFolder.Parent = game:GetService("CoreGui")

local espCache = {}  -- [playerName] = { highlight, billboard, ... }

--// Cria ESP pra um jogador
local function createESP(p)
	local char = p.Character
	if not char then return nil end

	-- Highlight
	local highlight = Instance.new("Highlight")
	highlight.Name = "ESP_" .. p.Name
	highlight.Adornee = char
	highlight.DepthMode = (ESP.DepthMode == "AlwaysOnTop") and Enum.HighlightDepthMode.AlwaysOnTop or Enum.HighlightDepthMode.Occluded
	highlight.FillTransparency = ESP.Transparency / 100
	highlight.OutlineTransparency = ESP.OutlineEnabled and 0 or 1
	highlight.FillColor = getESPColor(p)
	highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
	highlight.Parent = espFolder

	-- Billboard
	local billboard = Instance.new("BillboardGui")
	billboard.Name = "ESP_BB_" .. p.Name
	billboard.Adornee = char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart") or char
	billboard.Size = UDim2.fromOffset(220, 70)
	billboard.StudsOffset = Vector3.new(0, 3, 0)
	billboard.AlwaysOnTop = (ESP.DepthMode == "AlwaysOnTop")
	billboard.Parent = espFolder

	-- Nome
	local nameLabel = Instance.new("TextLabel")
	nameLabel.Name = "NameLabel"
	nameLabel.BackgroundTransparency = 1
	nameLabel.Size = UDim2.new(1, 0, 0, 16)
	nameLabel.Font = Enum.Font.GothamBold
	nameLabel.TextSize = 12
	nameLabel.TextColor3 = Color3.fromRGB(245, 245, 245)
	nameLabel.TextStrokeTransparency = 0
	nameLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	nameLabel.Text = p.DisplayName
	nameLabel.Visible = ESP.ShowName
	nameLabel.Parent = billboard

	-- Distância
	local distLabel = Instance.new("TextLabel")
	distLabel.Name = "DistLabel"
	distLabel.BackgroundTransparency = 1
	distLabel.Position = UDim2.new(0, 0, 0, 16)
	distLabel.Size = UDim2.new(1, 0, 0, 14)
	distLabel.Font = Enum.Font.Gotham
	distLabel.TextSize = 11
	distLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
	distLabel.TextStrokeTransparency = 0
	distLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	distLabel.Text = ""
	distLabel.Visible = ESP.ShowDistance
	distLabel.Parent = billboard

	-- Vida
	local healthLabel = Instance.new("TextLabel")
	healthLabel.Name = "HealthLabel"
	healthLabel.BackgroundTransparency = 1
	healthLabel.Position = UDim2.new(0, 0, 0, 30)
	healthLabel.Size = UDim2.new(1, 0, 0, 14)
	healthLabel.Font = Enum.Font.Gotham
	healthLabel.TextSize = 11
	healthLabel.TextColor3 = Color3.fromRGB(120, 220, 150)
	healthLabel.TextStrokeTransparency = 0
	healthLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	healthLabel.Text = ""
	healthLabel.Visible = ESP.ShowHealth
	healthLabel.Parent = billboard

	-- Target
	local targetLabel = Instance.new("TextLabel")
	targetLabel.Name = "TargetLabel"
	targetLabel.BackgroundTransparency = 1
	targetLabel.Position = UDim2.new(0, 0, 0, 44)
	targetLabel.Size = UDim2.new(1, 0, 0, 14)
	targetLabel.Font = Enum.Font.GothamBold
	targetLabel.TextSize = 11
	targetLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
	targetLabel.TextStrokeTransparency = 0
	targetLabel.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	targetLabel.Text = "🎯 TARGET"
	targetLabel.Visible = ESP.ShowTarget
	targetLabel.Parent = billboard

	return {
		highlight = highlight,
		billboard = billboard,
		nameLabel = nameLabel,
		distLabel = distLabel,
		healthLabel = healthLabel,
		targetLabel = targetLabel,
		char = char,
	}
end

--// Destrói ESP de um jogador
local function destroyESP(pName)
	local data = espCache[pName]
	if not data then return end
	if data.highlight then data.highlight:Destroy() end
	if data.billboard then data.billboard:Destroy() end
	espCache[pName] = nil
end

--// Atualiza ESP existente (recria se o character mudou)
local function updateESP(p)
	if not shouldShowESP(p) then
		destroyESP(p.Name)
		return
	end

	local data = espCache[p.Name]

	-- Se não existe ou o character mudou, recria
	if not data or data.char ~= p.Character then
		destroyESP(p.Name)
		local newData = createESP(p)
		if newData then
			espCache[p.Name] = newData
		end
		return
	end

	-- Atualiza cor
	if data.highlight and data.highlight.Parent then
		data.highlight.FillColor = getESPColor(p)
		data.highlight.FillTransparency = ESP.Transparency / 100
		data.highlight.OutlineTransparency = ESP.OutlineEnabled and 0 or 1
		data.highlight.DepthMode = (ESP.DepthMode == "AlwaysOnTop") and Enum.HighlightDepthMode.AlwaysOnTop or Enum.HighlightDepthMode.Occluded
	end

	-- Atualiza billboard
	if data.billboard and data.billboard.Parent then
		-- Nome
		if data.nameLabel then
			data.nameLabel.Visible = ESP.ShowName
			data.nameLabel.Text = p.DisplayName
		end

		-- Distância
		if data.distLabel then
			data.distLabel.Visible = ESP.ShowDistance
			if ESP.ShowDistance then
				local myChar = player.Character
				if myChar and myChar:FindFirstChild("HumanoidRootPart") and p.Character:FindFirstChild("HumanoidRootPart") then
					local dist = math.floor((myChar.HumanoidRootPart.Position - p.Character.HumanoidRootPart.Position).Magnitude)
					data.distLabel.Text = dist .. " studs"
				end
			end
		end

		-- Vida
		if data.healthLabel then
			data.healthLabel.Visible = ESP.ShowHealth
			if ESP.ShowHealth and p.Character then
				local hum = p.Character:FindFirstChildOfClass("Humanoid")
				if hum then
					local pct = math.floor((hum.Health / hum.MaxHealth) * 100)
					data.healthLabel.Text = "❤️ " .. pct .. "%"
					if pct > 60 then
						data.healthLabel.TextColor3 = Color3.fromRGB(120, 220, 150)
					elseif pct > 30 then
						data.healthLabel.TextColor3 = Color3.fromRGB(255, 200, 80)
					else
						data.healthLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
					end
				end
			end
		end

		-- Target
		if data.targetLabel then
			data.targetLabel.Visible = ESP.ShowTarget and isAimTarget(p)
		end

		-- Atualiza adornee
		if p.Character and data.billboard.Adornee ~= p.Character:FindFirstChild("Head") then
			data.billboard.Adornee = p.Character:FindFirstChild("Head") or p.Character:FindFirstChild("HumanoidRootPart") or p.Character
		end
	end
end
--// ============================================================
--// ESP — Loop principal com taxa de atualização
// ============================================================

local lastUpdate = 0

--// Limpa jogadores desconectados
local function cleanDisconnected()
	local currentNames = {}
	for _, p in ipairs(Players:GetPlayers()) do
		currentNames[p.Name] = true
	end

	for name in pairs(espCache) do
		if not currentNames[name] then
			destroyESP(name)
		end
	end
end

--// Loop de render
RunService.RenderStepped:Connect(function()
	local now = tick()
	if now - lastUpdate < ESP.RefreshRate then return end
	lastUpdate = now

	if not ESP.Enabled then
		-- Se desativou, limpa tudo
		if next(espCache) then
			for name in pairs(espCache) do
				destroyESP(name)
			end
		end
		return
	end

	cleanDisconnected()

	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= player then
			updateESP(p)
		end
	end
end)

--// Novo jogador entrou
Players.PlayerAdded:Connect(function(p)
	task.wait(0.5)
	if ESP.Enabled and p ~= player then
		updateESP(p)
	end
end)

--// Jogador saiu
Players.PlayerRemoving:Connect(function(p)
	destroyESP(p.Name)
end)

--// Novo character (respawn)
local function hookCharacter(p)
	p.CharacterAdded:Connect(function(char)
		destroyESP(p.Name)
		task.wait(0.5)
		if ESP.Enabled then
			updateESP(p)
		end
	end)
end

for _, p in ipairs(Players:GetPlayers()) do
	if p ~= player then
		hookCharacter(p)
	end
end

Players.PlayerAdded:Connect(function(p)
	if p ~= player then
		hookCharacter(p)
	end
end)
--// ============================================================
--// ESP — Interface, Registro e Load Automático
--// ============================================================

local function buildESPContent(container, ctx)
	-- Título
	local title = Instance.new("TextLabel")
	title.BackgroundTransparency = 1
	title.Position = UDim2.fromOffset(9, 6)
	title.Size = UDim2.new(1, -18, 0, 16)
	title.Font = Enum.Font.GothamBlack
	title.Text = "ESP"
	title.TextSize = 13
	title.TextColor3 = ctx.colors.TEXT
	title.TextXAlignment = Enum.TextXAlignment.Left
	title.Parent = container

	-- Scroll
	local scroll = Instance.new("ScrollingFrame")
	scroll.Size = UDim2.new(1, -14, 1, -28)
	scroll.Position = UDim2.fromOffset(7, 24)
	scroll.BackgroundTransparency = 1
	scroll.BorderSizePixel = 0
	scroll.ScrollBarThickness = 3
	scroll.ScrollBarImageColor3 = ctx.colors.ACCENT
	scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
	scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
	scroll.Parent = container

	local layout = Instance.new("UIListLayout")
	layout.Padding = UDim.new(0, 4)
	layout.SortOrder = Enum.SortOrder.LayoutOrder
	layout.Parent = scroll

	-- Helpers
	local function makeToggle(labelText, getter, setter)
		local row = Instance.new("Frame")
		row.Size = UDim2.new(1, -8, 0, 26)
		row.BackgroundColor3 = ctx.colors.CARD
		row.BorderSizePixel = 0
		row.Parent = scroll
		Instance.new("UICorner", row).CornerRadius = UDim.new(0, 5)

		local label = Instance.new("TextLabel")
		label.BackgroundTransparency = 1
		label.Position = UDim2.fromOffset(8, 0)
		label.Size = UDim2.new(1, -50, 1, 0)
		label.Font = Enum.Font.Gotham
		label.Text = labelText
		label.TextSize = 10
		label.TextColor3 = ctx.colors.TEXT
		label.TextXAlignment = Enum.TextXAlignment.Left
		label.Parent = row

		local btn = Instance.new("TextButton")
		btn.AnchorPoint = Vector2.new(1, 0.5)
		btn.Position = UDim2.new(1, -6, 0.5, 0)
		btn.Size = UDim2.fromOffset(34, 18)
		btn.BackgroundColor3 = getter() and ctx.colors.ACCENT or ctx.colors.PANEL
		btn.Text = ""
		btn.AutoButtonColor = false
		btn.Parent = row
		Instance.new("UICorner", btn).CornerRadius = UDim.new(1, 0)

		local knob = Instance.new("Frame")
		knob.Size = UDim2.fromOffset(14, 14)
		knob.Position = getter() and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
		knob.BackgroundColor3 = getter() and ctx.colors.BLACK or ctx.colors.SUBTEXT
		knob.BorderSizePixel = 0
		knob.Parent = btn
		Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

		btn.MouseButton1Click:Connect(function()
			local state = not getter()
			setter(state)
			btn.BackgroundColor3 = state and ctx.colors.ACCENT or ctx.colors.PANEL
			knob.BackgroundColor3 = state and ctx.colors.BLACK or ctx.colors.SUBTEXT
			knob.Position = state and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
		end)
	end

	local function makeSlider(labelText, min, max, getter, setter, onUpdate, suffix)
		local wrap = Instance.new("Frame")
		wrap.Size = UDim2.new(1, -8, 0, 36)
		wrap.BackgroundColor3 = ctx.colors.CARD
		wrap.BorderSizePixel = 0
		wrap.Parent = scroll
		Instance.new("UICorner", wrap).CornerRadius = UDim.new(0, 5)

		local label = Instance.new("TextLabel")
		label.BackgroundTransparency = 1
		label.Position = UDim2.fromOffset(8, 3)
		label.Size = UDim2.new(0.7, -10, 0, 12)
		label.Font = Enum.Font.Gotham
		label.Text = labelText
		label.TextSize = 10
		label.TextColor3 = ctx.colors.TEXT
		label.TextXAlignment = Enum.TextXAlignment.Left
		label.Parent = wrap

		local valueLabel = Instance.new("TextLabel")
		valueLabel.BackgroundTransparency = 1
		valueLabel.AnchorPoint = Vector2.new(1, 0)
		valueLabel.Position = UDim2.new(1, -8, 0, 3)
		valueLabel.Size = UDim2.new(0.3, 0, 0, 12)
		valueLabel.Font = Enum.Font.GothamBold
		valueLabel.Text = tostring(getter()) .. (suffix or "")
		valueLabel.TextSize = 10
		valueLabel.TextColor3 = ctx.colors.ACCENT
		valueLabel.TextXAlignment = Enum.TextXAlignment.Right
		valueLabel.Parent = wrap

		local track = Instance.new("Frame")
		track.Position = UDim2.fromOffset(8, 22)
		track.Size = UDim2.new(1, -16, 0, 5)
		track.BackgroundColor3 = ctx.colors.PANEL
		track.BorderSizePixel = 0
		track.Parent = wrap
		Instance.new("UICorner", track).CornerRadius = UDim.new(1, 0)

		local fraction = (getter() - min) / (max - min)
		local fill = Instance.new("Frame")
		fill.Size = UDim2.new(fraction, 0, 1, 0)
		fill.BackgroundColor3 = ctx.colors.ACCENT
		fill.BorderSizePixel = 0
		fill.Parent = track
		Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)

		local knob = Instance.new("Frame")
		knob.AnchorPoint = Vector2.new(0.5, 0.5)
		knob.Position = UDim2.new(fraction, 0, 0.5, 0)
		knob.Size = UDim2.fromOffset(11, 11)
		knob.BackgroundColor3 = ctx.colors.TEXT
		knob.BorderSizePixel = 0
		knob.Parent = track
		Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

		local dragging = false
		local function setFromX(x)
			local rel = math.clamp((x - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
			fill.Size = UDim2.new(rel, 0, 1, 0)
			knob.Position = UDim2.new(rel, 0, 0.5, 0)
			local v = math.floor(min + rel * (max - min) + 0.5)
			valueLabel.Text = tostring(v) .. (suffix or "")
			setter(v)
			if onUpdate then onUpdate() end
		end

		track.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
				dragging = true
				setFromX(input.Position.X)
			end
		end)
		knob.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
				dragging = true
			end
		end)
		game:GetService("UserInputService").InputChanged:Connect(function(input)
			if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
				setFromX(input.Position.X)
			end
		end)
		game:GetService("UserInputService").InputEnded:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
				dragging = false
			end
		end)
	end

	local function makeDropdown(labelText, options, getter, setter, onUpdate)
		local wrap = Instance.new("Frame")
		wrap.Size = UDim2.new(1, -8, 0, 40)
		wrap.BackgroundColor3 = ctx.colors.CARD
		wrap.BorderSizePixel = 0
		wrap.Parent = scroll
		Instance.new("UICorner", wrap).CornerRadius = UDim.new(0, 5)

		local label = Instance.new("TextLabel")
		label.BackgroundTransparency = 1
		label.Position = UDim2.fromOffset(8, 3)
		label.Size = UDim2.new(1, -16, 0, 12)
		label.Font = Enum.Font.Gotham
		label.Text = labelText
		label.TextSize = 10
		label.TextColor3 = ctx.colors.TEXT
		label.TextXAlignment = Enum.TextXAlignment.Left
		label.Parent = wrap

		local btn = Instance.new("TextButton")
		btn.Position = UDim2.fromOffset(8, 18)
		btn.Size = UDim2.new(1, -16, 0, 16)
		btn.BackgroundColor3 = ctx.colors.PANEL
		btn.Text = "  " .. getter() .. "  ▼"
		btn.Font = Enum.Font.Gotham
		btn.TextSize = 10
		btn.TextColor3 = ctx.colors.TEXT
		btn.TextXAlignment = Enum.TextXAlignment.Left
		btn.AutoButtonColor = false
		btn.Parent = wrap
		Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)

		local expanded = false
		local optionFrame

		btn.MouseButton1Click:Connect(function()
			expanded = not expanded
			if expanded then
				optionFrame = Instance.new("Frame")
				optionFrame.Size = UDim2.new(1, 0, 0, #options * 18 + 6)
				optionFrame.Position = UDim2.new(0, 0, 1, 2)
				optionFrame.BackgroundColor3 = ctx.colors.PANEL
				optionFrame.BorderSizePixel = 0
				optionFrame.ZIndex = 20
				optionFrame.Parent = wrap
				Instance.new("UICorner", optionFrame).CornerRadius = UDim.new(0, 4)

				local optLayout = Instance.new("UIListLayout")
				optLayout.Padding = UDim.new(0, 1)
				optLayout.Parent = optionFrame

				local pad = Instance.new("UIPadding")
				pad.PaddingLeft = UDim.new(0, 3)
				pad.PaddingTop = UDim.new(0, 3)
				pad.PaddingRight = UDim.new(0, 3)
				pad.PaddingBottom = UDim.new(0, 3)
				pad.Parent = optionFrame

				for _, opt in ipairs(options) do
					local ob = Instance.new("TextButton")
					ob.Size = UDim2.new(1, 0, 0, 16)
					ob.BackgroundTransparency = 1
					ob.Text = "  " .. opt
					ob.Font = Enum.Font.Gotham
					ob.TextSize = 9
					ob.TextColor3 = ctx.colors.TEXT
					ob.TextXAlignment = Enum.TextXAlignment.Left
					ob.AutoButtonColor = false
					ob.ZIndex = 21
					ob.Parent = optionFrame

					ob.MouseEnter:Connect(function()
						ob.BackgroundTransparency = 0
						ob.BackgroundColor3 = ctx.colors.CARD
					end)
					ob.MouseLeave:Connect(function()
						ob.BackgroundTransparency = 1
					end)
					ob.MouseButton1Click:Connect(function()
						setter(opt)
						btn.Text = "  " .. opt .. "  ▼"
						expanded = false
						if optionFrame then optionFrame:Destroy() end
						if onUpdate then onUpdate() end
					end)
				end
			else
				if optionFrame then optionFrame:Destroy() end
			end
		end)
	end

	-- Controles
	makeToggle("Ativar ESP", function() return ESP.Enabled end, function(v) ESP.Enabled = v; persist() end)
	makeToggle("ESP por Time", function() return ESP.TeamCheck end, function(v) ESP.TeamCheck = v; persist() end)
	makeToggle("Mostrar Amigos", function() return ESP.ShowFriends end, function(v) ESP.ShowFriends = v; persist() end)
	makeToggle("Só Amigos (conexões)", function() return ESP.OnlyFriends end, function(v) ESP.OnlyFriends = v; persist() end)
	makeToggle("Esconder Mortos", function() return ESP.HideDead end, function(v) ESP.HideDead = v; persist() end)
	makeToggle("Mostrar Distância", function() return ESP.ShowDistance end, function(v) ESP.ShowDistance = v; persist() end)
	makeToggle("Mostrar Nome", function() return ESP.ShowName end, function(v) ESP.ShowName = v; persist() end)
	makeToggle("Mostrar Vida", function() return ESP.ShowHealth end, function(v) ESP.ShowHealth = v; persist() end)
	makeToggle("Mostrar Alvo (Target)", function() return ESP.ShowTarget end, function(v) ESP.ShowTarget = v; persist() end)
	makeToggle("Mostrar Contorno", function() return ESP.OutlineEnabled end, function(v) ESP.OutlineEnabled = v; persist() end)

	makeSlider("Distância Máxima", 100, 5000, function() return ESP.MaxDistance end, function(v) ESP.MaxDistance = v; persist() end, nil, " studs")
	makeSlider("Transparência", 0, 100, function() return ESP.Transparency end, function(v) ESP.Transparency = v; persist() end, nil, "%")
	makeSlider("Taxa de Atualização", 1, 50, function() return math.floor(ESP.RefreshRate * 100) end, function(v) ESP.RefreshRate = v / 100; persist() end, nil, "ms")

	makeDropdown("Cor (Inimigos)", {"Branco", "Vermelho", "Verde", "Azul", "Amarelo", "Roxo", "Rosa", "Ciano", "Laranja"}, function() return ESP.ColorEnemy end, function(v) ESP.ColorEnemy = v; persist() end)
	makeDropdown("Cor (Time)", {"Branco", "Vermelho", "Verde", "Azul", "Amarelo", "Roxo", "Rosa", "Ciano", "Laranja"}, function() return ESP.ColorTeam end, function(v) ESP.ColorTeam = v; persist() end)
	makeDropdown("Cor (Amigos)", {"Branco", "Vermelho", "Verde", "Azul", "Amarelo", "Roxo", "Rosa", "Ciano", "Laranja"}, function() return ESP.ColorFriend end, function(v) ESP.ColorFriend = v; persist() end)

	makeDropdown("Modo de Profundidade", {"AlwaysOnTop", "Occluded"}, function() return ESP.DepthMode end, function(v) ESP.DepthMode = v; persist() end)
end

--// REGISTRO — ESP (sem ícone por enquanto)
api:Invoke("Batata001", {
	PluginId = "Batata012",
	Name = "ESP",
	IconId = nil,
	BuildContent = buildESPContent,
})

print("[Módulo ESP v1] Registrado com PluginId: Batata012 (universal)")
