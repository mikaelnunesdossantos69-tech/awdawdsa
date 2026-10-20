--========================================================
-- N4T HUB
--========================================================

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local SoundService = game:GetService("SoundService")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local TextChatService = game:GetService("TextChatService")
local TextService = game:GetService("TextService")
local MaterialService = game:GetService("MaterialService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

--========================================================
-- COMPATIBILIDADE ENTRE EXECUTORES
--========================================================
local ExecutorCompat = {}

do
    local globalEnv = _G
    pcall(function()
        if type(getgenv) == "function" then
            local env = getgenv()
            if type(env) == "table" then
                globalEnv = env
            end
        end
    end)

    local compiler = loadstring
    if type(compiler) ~= "function" and type(globalEnv) == "table" then
        compiler = globalEnv.loadstring
    end
    if type(compiler) ~= "function" and type(load) == "function" then
        compiler = load
    end

    function ExecutorCompat.compile(source, chunkName)
        if type(compiler) ~= "function" then
            return nil, "Este executor não disponibiliza loadstring/load."
        end

        local ok, fn, err = pcall(compiler, source, chunkName)
        if not ok then
            return nil, tostring(fn)
        end
        if type(fn) ~= "function" then
            return nil, tostring(err or "Falha ao compilar o código.")
        end
        return fn
    end

    local function tryRequest(req, url)
        if type(req) ~= "function" then
            return nil
        end

        local ok, response = pcall(req, {
            Url = url,
            URL = url,
            Method = "GET",
            method = "GET"
        })

        if not ok then
            return nil
        end

        if type(response) == "string" then
            return response
        end

        if type(response) == "table" then
            local body = response.Body or response.body or response.ResponseBody or response.responseBody
            if type(body) == "string" and #body > 0 then
                return body
            end
        end

        return nil
    end

    function ExecutorCompat.httpGet(url)
        local ok, body = pcall(function()
            return game:HttpGet(url)
        end)

        if ok and type(body) == "string" and #body > 0 then
            return body
        end

        local candidates = {}

        pcall(function()
            if type(request) == "function" then table.insert(candidates, request) end
        end)
        pcall(function()
            if type(http_request) == "function" then table.insert(candidates, http_request) end
        end)
        pcall(function()
            if type(syn) == "table" and type(syn.request) == "function" then
                table.insert(candidates, syn.request)
            end
        end)
        pcall(function()
            if type(fluxus) == "table" and type(fluxus.request) == "function" then
                table.insert(candidates, fluxus.request)
            end
        end)

        if type(globalEnv) == "table" then
            pcall(function()
                if type(globalEnv.request) == "function" then
                    table.insert(candidates, globalEnv.request)
                end
            end)
            pcall(function()
                if type(globalEnv.http_request) == "function" then
                    table.insert(candidates, globalEnv.http_request)
                end
            end)
        end

        for _, req in ipairs(candidates) do
            local result = tryRequest(req, url)
            if result then
                return result
            end
        end

        error("O executor não liberou HTTP GET para este script.")
    end

    pcall(function()
        if type(globalEnv) == "table" then
            if type(globalEnv.loadstring) ~= "function" and type(compiler) == "function" then
                globalEnv.loadstring = compiler
            end
            if type(globalEnv.N4T_HTTPGET) ~= "function" then
                globalEnv.N4T_HTTPGET = ExecutorCompat.httpGet
            end
        end
    end)
end

--========================================================
-- N4T HUB UI - COMPAT / HORIZONTAL / CATEGORIAS / NEON
-- A interface fica isolada em uma tabela para evitar excesso de
-- variaveis locais no chunk principal em executores mais limitados.
--========================================================

local RED = Color3.fromRGB(0, 205, 255)
local BG = Color3.fromRGB(3, 6, 11)
local OFF_COLOR = Color3.fromRGB(10, 17, 27)
local ON_COLOR = Color3.fromRGB(7, 48, 66)
local TEXT = Color3.fromRGB(255, 255, 255)
local TEXT_OFF = Color3.fromRGB(155, 185, 210)

local UI = {}

do
    local parent = playerGui

    -- Alguns executores preferem gethui/CoreGui. Se nao existir,
    -- PlayerGui continua sendo usado normalmente.
    pcall(function()
        if type(gethui) == "function" then
            local h = gethui()
            if h then parent = h end
        end
    end)

    local function destroyOld(container)
        if not container then return end
        local old = container:FindFirstChild("N4T_HUB")
            or container:FindFirstChild("MM2_RedHub")
            or container:FindFirstChild("MM2_CyanHub")
        if old then pcall(function() old:Destroy() end) end
    end

    destroyOld(playerGui)
    if parent ~= playerGui then destroyOld(parent) end

    local gui = Instance.new("ScreenGui")
    gui.Name = "N4T_HUB"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    pcall(function() gui.DisplayOrder = 999 end)
    gui.Parent = parent
    UI.gui = gui

    local camera = Workspace.CurrentCamera
    local viewport = camera and camera.ViewportSize or Vector2.new(800, 600)
    local width = viewport.X - 40
    local height = viewport.Y - 130
    if width < 320 then width = 320 end
    if width > 480 then width = 480 end
    if height < 260 then height = 260 end
    if height > 320 then height = 320 end

    UI.bigSize = UDim2.fromOffset(width, height)
    UI.smallSize = UDim2.fromOffset(150, 44)

    local shell = Instance.new("Frame")
    shell.Name = "Shell"
    shell.AnchorPoint = Vector2.new(0.5, 0.5)
    shell.Position = UDim2.fromScale(0.5, 0.5)
    shell.Size = UI.bigSize
    shell.BackgroundTransparency = 1
    shell.BorderSizePixel = 0
    shell.Active = true
    shell.ZIndex = 5
    shell.Parent = gui
    UI.shell = shell

    local main = Instance.new("Frame")
    main.Name = "Main"
    main.Size = UDim2.fromScale(1, 1)
    main.BackgroundColor3 = BG
    main.BorderSizePixel = 0
    main.ClipsDescendants = true
    main.Active = true
    main.ZIndex = 10
    main.Parent = shell
    UI.main = main

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 20)
    corner.Parent = main

    local stroke = Instance.new("UIStroke")
    stroke.Color = RED
    stroke.Thickness = 2
    stroke.Transparency = 0.08
    stroke.Parent = main
    UI.mainStroke = stroke

    pcall(function()
        local gradient = Instance.new("UIGradient")
        gradient.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(2, 5, 10)),
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(8, 16, 28)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(3, 7, 12))
        })
        gradient.Rotation = 10
        gradient.Parent = main
        UI.mainGradient = gradient
    end)

    UI.glowStrokes = {}
    local function addGlow(extra, thickness, transparency, color)
        local g = Instance.new("Frame")
        g.Name = "NeonGlow"
        g.Position = UDim2.fromOffset(-extra, -extra)
        g.Size = UDim2.new(1, extra * 2, 1, extra * 2)
        g.BackgroundTransparency = 1
        g.BorderSizePixel = 0
        g.ZIndex = 4
        g.Parent = shell

        local gc = Instance.new("UICorner")
        gc.CornerRadius = UDim.new(0, 20 + extra)
        gc.Parent = g

        local gs = Instance.new("UIStroke")
        gs.Color = color
        gs.Thickness = thickness
        gs.Transparency = transparency
        gs.Parent = g
        table.insert(UI.glowStrokes, gs)
    end

    pcall(function()
        addGlow(4, 2, 0.28, RED)
        addGlow(8, 3, 0.58, RED)
        addGlow(13, 5, 0.80, Color3.fromRGB(90, 80, 255))
    end)

    local header = Instance.new("TextButton")
    header.Name = "DragHeader"
    header.Size = UDim2.new(1, 0, 0, 62)
    header.BackgroundTransparency = 1
    header.Text = ""
    header.AutoButtonColor = false
    header.Active = true
    header.ZIndex = 30
    header.Parent = main
    UI.header = header

    local logo = Instance.new("Frame")
    logo.Size = UDim2.fromOffset(42, 42)
    logo.Position = UDim2.fromOffset(12, 10)
    logo.BackgroundColor3 = RED
    logo.BorderSizePixel = 0
    logo.ZIndex = 31
    logo.Parent = header

    local logoCorner = Instance.new("UICorner")
    logoCorner.CornerRadius = UDim.new(0, 13)
    logoCorner.Parent = logo

    local logoText = Instance.new("TextLabel")
    logoText.Size = UDim2.fromScale(1, 1)
    logoText.BackgroundTransparency = 1
    logoText.Text = "N4T"
    logoText.TextColor3 = TEXT
    logoText.Font = Enum.Font.GothamBold
    logoText.TextSize = 13
    logoText.ZIndex = 32
    logoText.Parent = logo

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -130, 0, 25)
    title.Position = UDim2.fromOffset(65, 8)
    title.BackgroundTransparency = 1
    title.Text = "N4T HUB"
    title.TextColor3 = TEXT
    title.Font = Enum.Font.GothamBold
    title.TextSize = 17
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.ZIndex = 31
    title.Parent = header

    local subtitle = Instance.new("TextLabel")
    subtitle.Size = UDim2.new(1, -130, 0, 18)
    subtitle.Position = UDim2.fromOffset(65, 32)
    subtitle.BackgroundTransparency = 1
    subtitle.Text = "MM2 CONTROL CENTER"
    subtitle.TextColor3 = TEXT_OFF
    subtitle.Font = Enum.Font.Gotham
    subtitle.TextSize = 10
    subtitle.TextXAlignment = Enum.TextXAlignment.Left
    subtitle.ZIndex = 31
    subtitle.Parent = header
    UI.subtitle = subtitle

    local minus = Instance.new("TextButton")
    minus.Name = "Minimize"
    minus.Size = UDim2.fromOffset(38, 34)
    minus.Position = UDim2.new(1, -50, 0, 13)
    minus.BackgroundColor3 = Color3.fromRGB(9, 24, 35)
    minus.BorderSizePixel = 0
    minus.Text = "-"
    minus.TextColor3 = RED
    minus.Font = Enum.Font.GothamBold
    minus.TextSize = 20
    minus.AutoButtonColor = false
    minus.Visible = true
    minus.ZIndex = 40
    minus.Parent = main
    UI.minus = minus

    local minusCorner = Instance.new("UICorner")
    minusCorner.CornerRadius = UDim.new(0, 11)
    minusCorner.Parent = minus

    local waveLine = Instance.new("Frame")
    waveLine.Name = "WaveLine"
    waveLine.Size = UDim2.new(1, -24, 0, 2)
    waveLine.Position = UDim2.fromOffset(12, 61)
    waveLine.BackgroundColor3 = RED
    waveLine.BorderSizePixel = 0
    waveLine.ZIndex = 25
    waveLine.Parent = main
    UI.waveLine = waveLine

    local waveGlow = Instance.new("Frame")
    waveGlow.Name = "WaveGlow"
    waveGlow.AnchorPoint = Vector2.new(0.5, 0.5)
    waveGlow.Size = UDim2.fromOffset(90, 5)
    waveGlow.Position = UDim2.new(0, -45, 0.5, 0)
    waveGlow.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    waveGlow.BackgroundTransparency = 0.15
    waveGlow.BorderSizePixel = 0
    waveGlow.ZIndex = 26
    waveGlow.Parent = waveLine
    UI.waveGlow = waveGlow

    local categoryBar = Instance.new("ScrollingFrame")
    categoryBar.Name = "Categories"
    categoryBar.Size = UDim2.new(1, -24, 0, 40)
    categoryBar.Position = UDim2.fromOffset(12, 68)
    categoryBar.BackgroundTransparency = 1
    categoryBar.BorderSizePixel = 0
    categoryBar.ScrollBarThickness = 0
    categoryBar.ScrollingDirection = Enum.ScrollingDirection.X
    categoryBar.AutomaticCanvasSize = Enum.AutomaticSize.X
    categoryBar.CanvasSize = UDim2.new()
    categoryBar.ZIndex = 20
    categoryBar.Parent = main
    UI.categoryBar = categoryBar

    local categoryLayout = Instance.new("UIListLayout")
    categoryLayout.FillDirection = Enum.FillDirection.Horizontal
    categoryLayout.HorizontalAlignment = Enum.HorizontalAlignment.Left
    categoryLayout.VerticalAlignment = Enum.VerticalAlignment.Center
    categoryLayout.Padding = UDim.new(0, 8)
    categoryLayout.SortOrder = Enum.SortOrder.LayoutOrder
    categoryLayout.Parent = categoryBar
    UI.categoryLayout = categoryLayout

    local content = Instance.new("ScrollingFrame")
    content.Name = "Scripts"
    content.Size = UDim2.new(1, -24, 1, -120)
    content.Position = UDim2.fromOffset(12, 114)
    content.BackgroundTransparency = 1
    content.BorderSizePixel = 0
    content.ScrollBarThickness = 3
    content.ScrollBarImageColor3 = RED
    content.AutomaticCanvasSize = Enum.AutomaticSize.Y
    content.CanvasSize = UDim2.new()
    content.Active = true
    content.Visible = true
    content.ZIndex = 15
    content.Parent = main
    UI.content = content

    local padding = Instance.new("UIPadding")
    padding.PaddingTop = UDim.new(0, 2)
    padding.PaddingBottom = UDim.new(0, 8)
    padding.PaddingLeft = UDim.new(0, 1)
    padding.PaddingRight = UDim.new(0, 1)
    padding.Parent = content

    local grid = Instance.new("UIGridLayout")
    grid.CellPadding = UDim2.fromOffset(8, 8)
    grid.CellSize = UDim2.new(0.5, -5, 0, 52)
    grid.FillDirection = Enum.FillDirection.Horizontal
    grid.FillDirectionMaxCells = 2
    grid.HorizontalAlignment = Enum.HorizontalAlignment.Center
    grid.SortOrder = Enum.SortOrder.LayoutOrder
    grid.Parent = content
    UI.grid = grid
end

--========================================================
-- ESTADOS
--========================================================

local states = {
    ["ANTI INVALID"] = false,
    ["ANIM ANTIGA"] = false,
    ["CONFETE"] = false,
    ["SOM DA ARMA"] = false,
    ["SPEED"] = false,
    ["SPEED RAW"] = false,
    ["SPEED CWLLES"] = false,
    ["SPEED 1.2"] = false,
    ["BALAO DE CHAT OLD"] = false,
    ["CORPO ANTIGO"] = false,
    ["KORBLOX BIEL"] = false,
    ["2023 XP"] = false,
    ["HOT BAR 2025"] = false,
    ["2023 SHIFT"] = false,
    ["2024 SHIFT"] = false,
    ["HOT BAR TEXT"] = false,
    ["2020 QUALITY"] = false,
    ["2023 QUALITY"] = false,
    ["WATERBALLOON"] = false,
    ["TIRAR X DA GUN"] = false,
    ["VOICE 2023"] = false,
    ["SNOWBALL"] = false,
    ["HEADLESS"] = false,
    ["SPEED COIL"] = false,
    ["GRAVITY COIL"] = false,
    ["SOMBRAS"] = false,
    ["SOM ABAFADO"] = false,
    ["VANGUARD HUB"] = false,
    ["PSHADE"] = false,
    ["NÚMEROS PEQUENOS"] = false,
    ["SPEED OLD"] = false,
    ["OLD SPEED NICKY"] = false,
    ["SPEED EMOTE"] = false,
    ["SPEED DOUGLAS"] = false,
    ["SG NATAN"] = false,
    ["ANTI CLICK"] = false,
    ["MOV GUN"] = false,
    ["JUMP WATER"] = false,
    ["FOG HALLOWEEN"] = false
}

local buttons = {}
local updateButton

--========================================================
-- BALAO DE CHAT OLD
--========================================================

local chatOldConnections = {}
local baloesChat = {}

local TEMPO_NA_TELA = 8
local MAX_BALOES = 4
local ALTURA_BASE = 1.83
local DISTANCIA_ENTRE_BALOES = 0.83
local DISTANCIA_MAXIMA = 100
local DISTANCIA_PONTINHOS = 45

local function calcularTamanhoChat(texto)
    local bounds = TextService:GetTextSize(texto, 20, Enum.Font.SourceSans, Vector2.new(220, 1000))
    local largura = math.clamp(bounds.X + 16, 36, 240)
    local altura = math.clamp(bounds.Y + 8, 30, 120)
    return largura, altura
end

local function reorganizarChat(plr)
    local lista = baloesChat[plr]
    if not lista then return end

    for i, guiObj in ipairs(lista) do
        if guiObj and guiObj.Parent then
            local distanciaDoNovo = #lista - i
            guiObj.StudsOffsetWorldSpace = Vector3.new(0, ALTURA_BASE + (distanciaDoNovo * DISTANCIA_ENTRE_BALOES), 0)

            local balao = guiObj:FindFirstChild("Balao")
            if balao then
                local ponta = balao:FindFirstChild("Ponta")
                if ponta then ponta.Visible = (i == #lista) end
            end
        end
    end
end

local function removerBalaoChat(plr, guiObj)
    local lista = baloesChat[plr]
    if not lista then return end

    for i, item in ipairs(lista) do
        if item == guiObj then
            table.remove(lista, i)
            break
        end
    end

    if guiObj and guiObj.Parent then guiObj:Destroy() end
    reorganizarChat(plr)
end

local function criarBalaoChat(plr, texto)
    if not states["BALAO DE CHAT OLD"] then return end
    local character = plr.Character
    if not character then return end
    local head = character:FindFirstChild("Head")
    if not head then return end

    baloesChat[plr] = baloesChat[plr] or {}
    local lista = baloesChat[plr]

    while #lista >= MAX_BALOES do
        local antigo = table.remove(lista, 1)
        if antigo then antigo:Destroy() end
    end

    local largura, altura = calcularTamanhoChat(texto)

    local guiObj = Instance.new("BillboardGui")
    guiObj.Name = "BalaoChatClassico"
    guiObj.Adornee = head
    guiObj.Size = UDim2.fromOffset(largura, altura + 12)
    guiObj.AlwaysOnTop = false
    guiObj.MaxDistance = DISTANCIA_MAXIMA
    guiObj.Parent = head

    local balao = Instance.new("Frame")
    balao.Name = "Balao"
    balao.AnchorPoint = Vector2.new(0.5, 0)
    balao.Position = UDim2.new(0.5, 0, 0, 0)
    balao.Size = UDim2.fromOffset(largura, altura)
    balao.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    balao.BorderSizePixel = 0
    balao.ZIndex = 2
    balao.Parent = guiObj

    local canto = Instance.new("UICorner")
    canto.CornerRadius = UDim.new(0, 4)
    canto.Parent = balao

    local label = Instance.new("TextLabel")
    label.Name = "Texto"
    label.BackgroundTransparency = 1
    label.Size = UDim2.new(1, -8, 1, 0)
    label.Position = UDim2.new(0, 4, 0, 0)
    label.Text = texto
    label.TextColor3 = Color3.fromRGB(57, 68, 76)
    label.TextSize = 20
    label.Font = Enum.Font.SourceSans
    label.TextWrapped = true
    label.TextXAlignment = Enum.TextXAlignment.Center
    label.TextYAlignment = Enum.TextYAlignment.Center
    label.ZIndex = 5
    label.Parent = balao

    local ponta = Instance.new("Frame")
    ponta.Name = "Ponta"
    ponta.AnchorPoint = Vector2.new(0.5, 0.5)
    ponta.Position = UDim2.new(0.5, 0, 1, 0)
    ponta.Size = UDim2.fromOffset(16, 14)
    ponta.Rotation = 45
    ponta.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ponta.BorderSizePixel = 0
    ponta.ZIndex = 1
    ponta.Parent = balao

    local conexaoDistancia
    conexaoDistancia = RunService.RenderStepped:Connect(function()
        if not guiObj or not guiObj.Parent or not head then
            conexaoDistancia:Disconnect()
            return
        end

        local camera = Workspace.CurrentCamera
        if not camera then return end

        local dist = (camera.CFrame.Position - head.Position).Magnitude

        if dist >= DISTANCIA_PONTINHOS then
            label.Text = "..."
            label.TextSize = 24
            balao.Size = UDim2.fromOffset(45, 28)
            guiObj.Size = UDim2.fromOffset(45, 40)
        else
            label.Text = texto
            label.TextSize = 20
            balao.Size = UDim2.fromOffset(largura, altura)
            guiObj.Size = UDim2.fromOffset(largura, altura + 12)
        end
    end)

    table.insert(lista, guiObj)
    reorganizarChat(plr)

    task.delay(TEMPO_NA_TELA, function()
        if conexaoDistancia then conexaoDistancia:Disconnect() end
        if guiObj and guiObj.Parent then removerBalaoChat(plr, guiObj) end
    end)
end

local function balaoChatOldOn()
    local bubbleConfig = TextChatService:FindFirstChildOfClass("BubbleChatConfiguration")
    if bubbleConfig then bubbleConfig.Enabled = false end

    table.insert(chatOldConnections, TextChatService.MessageReceived:Connect(function(message)
        if not message.TextSource then return end
        local plr = Players:GetPlayerByUserId(message.TextSource.UserId)
        if plr then criarBalaoChat(plr, message.Text) end
    end))

    table.insert(chatOldConnections, Players.PlayerRemoving:Connect(function(plr)
        baloesChat[plr] = nil
    end))

    print("✅ BALAO DE CHAT OLD ON")
end

local function balaoChatOldOff()
    for _, conn in ipairs(chatOldConnections) do
        pcall(function() conn:Disconnect() end)
    end
    chatOldConnections = {}

    for plr, lista in pairs(baloesChat) do
        for _, guiObj in ipairs(lista) do
            if guiObj and guiObj.Parent then guiObj:Destroy() end
        end
    end
    table.clear(baloesChat)

    local bubbleConfig = TextChatService:FindFirstChildOfClass("BubbleChatConfiguration")
    if bubbleConfig then bubbleConfig.Enabled = true end

    print("❌ BALAO DE CHAT OLD OFF")
end

--========================================================
-- SPEED RAW
--========================================================

local speedRawConnections = {}
local activeGhostRaw, activeWeldRaw
local currentOffsetRaw = CFrame.new(0, 0, 0)

local function deployGhostGlitchRaw()
    if not states["SPEED RAW"] then return end
    local char = player.Character
    if not char then return end

    local cachedRoot = char:FindFirstChild("HumanoidRootPart")
    local cachedTorso = char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
    if not cachedTorso or not cachedRoot then return end

    local rawCF = cachedRoot.CFrame:ToObjectSpace(cachedTorso.CFrame)
    currentOffsetRaw = CFrame.new(rawCF.Position * 1.9) * rawCF.Rotation

    local oldGhost = char:FindFirstChild("Skibidi_Ghost_Active")
    if oldGhost then oldGhost:Destroy() end

    local newGhost = Instance.new("Part")
    newGhost.Name = "Skibidi_Ghost_Active"
    newGhost.Size = Vector3.new(1, 1, 1)

    local targetMass = 13
    local volume = newGhost.Size.X * newGhost.Size.Y * newGhost.Size.Z
    local requiredDensity = targetMass / volume

    newGhost.CustomPhysicalProperties = PhysicalProperties.new(requiredDensity, 0.7, 0.3, 1, 1)
    newGhost.CFrame = cachedTorso.CFrame 
    newGhost.Transparency = 1 
    newGhost.CanCollide = false
    newGhost.Parent = char

    activeWeldRaw = Instance.new("Weld")
    activeWeldRaw.Part0 = cachedTorso
    activeWeldRaw.Part1 = newGhost
    activeWeldRaw.C0 = currentOffsetRaw
    activeWeldRaw.Parent = newGhost

    activeGhostRaw = newGhost
end

local function speedRawOn()
    if states["SPEED"] then states["SPEED"] = false; if offFunctions and offFunctions["SPEED"] then offFunctions["SPEED"]() end; updateButton("SPEED") end
    if states["SPEED CWLLES"] then states["SPEED CWLLES"] = false; if offFunctions and offFunctions["SPEED CWLLES"] then offFunctions["SPEED CWLLES"]() end; updateButton("SPEED CWLLES") end

    local function setupListeners(char)
        if not char then return end
        local humanoid = char:FindFirstChild("Humanoid")
        if humanoid then
            table.insert(speedRawConnections, humanoid.Died:Connect(function()
                activeGhostRaw = nil
                activeWeldRaw = nil
            end))
        end

        task.defer(deployGhostGlitchRaw)

        table.insert(speedRawConnections, char.ChildAdded:Connect(function(child)
            if not states["SPEED RAW"] then return end
            if child:IsA("Tool") or string.find(string.lower(child.Name), "radio") then 
                for _, part in ipairs(child:GetDescendants()) do
                    if part:IsA("BasePart") then part.Massless = true end
                end
                deployGhostGlitchRaw() 
            end
        end))

        table.insert(speedRawConnections, char.ChildRemoved:Connect(function(child)
            if not states["SPEED RAW"] then return end
            if child:IsA("Tool") or string.find(string.lower(child.Name), "radio") then 
                deployGhostGlitchRaw() 
            end
        end))
    end

    if player.Character then setupListeners(player.Character) end
    table.insert(speedRawConnections, player.CharacterAdded:Connect(setupListeners))

    table.insert(speedRawConnections, RunService.Heartbeat:Connect(function()
        if not states["SPEED RAW"] then return end
        if activeWeldRaw then
            activeWeldRaw.C0 = currentOffsetRaw
        else
            if player.Character and (not activeGhostRaw or activeGhostRaw.Parent ~= player.Character) then
                deployGhostGlitchRaw()
            end
        end
    end))

    print("✅ SPEED RAW ON")
end

local function speedRawOff()
    for _, conn in ipairs(speedRawConnections) do
        pcall(function() conn:Disconnect() end)
    end
    speedRawConnections = {}

    if activeGhostRaw then pcall(function() activeGhostRaw:Destroy() end) end
    activeGhostRaw = nil
    activeWeldRaw = nil

    print("❌ SPEED RAW OFF")
end

--========================================================
-- SPEED CWLLES
--========================================================

local speedCwllesConnections = {}

local function aplicarCwlles(character)
    if not character or not states["SPEED CWLLES"] then return end

    local humanoid = character:WaitForChild("Humanoid", 10)
    local rootPart = character:WaitForChild("HumanoidRootPart", 10)
    if not humanoid or not rootPart then return end

    if character:FindFirstChild("GlitchPart") then
        character.GlitchPart:Destroy()
    end

    local glitchPart = Instance.new("Part")
    glitchPart.Name = "GlitchPart"
    glitchPart.Size = Vector3.new(2, 2, 2)
    glitchPart.Transparency = 1
    glitchPart.CanCollide = false
    glitchPart.CanTouch = false
    glitchPart.CanQuery = false
    glitchPart.Anchored = false
    glitchPart.Massless = false
    glitchPart.Parent = character

    local weld = Instance.new("Weld")
    weld.Part0 = rootPart
    weld.Part1 = glitchPart
    weld.C0 = CFrame.new(0, 0, 0)
    weld.Parent = glitchPart

    local escalando = false

    local function setOffset(z) weld.C0 = CFrame.new(0, 0, z) end
    local function deslocarMassa() setOffset(10.5) end
    local function resetarMassa() setOffset(0) end

    table.insert(speedCwllesConnections, humanoid.StateChanged:Connect(function(_, newState)
        escalando = (newState == Enum.HumanoidStateType.Climbing)
    end))

    table.insert(speedCwllesConnections, character.ChildAdded:Connect(function(obj)
        if obj:IsA("Tool") then
            if escalando then deslocarMassa() else resetarMassa() end
        end
    end))

    table.insert(speedCwllesConnections, character.ChildRemoved:Connect(function(obj)
        if obj:IsA("Tool") and not escalando then resetarMassa() end
    end))

    local backpack = player:WaitForChild("Backpack", 5)
    if backpack then
        table.insert(speedCwllesConnections, backpack.ChildAdded:Connect(function(obj)
            if obj:IsA("Tool") and not escalando then resetarMassa() end
        end))
        table.insert(speedCwllesConnections, backpack.ChildRemoved:Connect(function(obj)
            if obj:IsA("Tool") and not escalando then resetarMassa() end
        end))
    end
end

local function speedCwllesOn()
    if states["SPEED"] then states["SPEED"] = false; if offFunctions and offFunctions["SPEED"] then offFunctions["SPEED"]() end; updateButton("SPEED") end
    if states["SPEED RAW"] then states["SPEED RAW"] = false; if offFunctions and offFunctions["SPEED RAW"] then offFunctions["SPEED RAW"]() end; updateButton("SPEED RAW") end

    if player.Character then aplicarCwlles(player.Character) end
    table.insert(speedCwllesConnections, player.CharacterAdded:Connect(aplicarCwlles))

    print("✅ SPEED CWLLES ON")
end

local function speedCwllesOff()
    for _, conn in ipairs(speedCwllesConnections) do
        pcall(function() conn:Disconnect() end)
    end
    speedCwllesConnections = {}

    if player.Character and player.Character:FindFirstChild("GlitchPart") then
        pcall(function() player.Character.GlitchPart:Destroy() end)
    end

    print("❌ SPEED CWLLES OFF")
end

--========================================================
-- 2023 QUALITY
--========================================================

local quality2023Connections = {}
local originalLightBrightness = {}

local function ajustarLuzes2023()
    for _, v in pairs(Workspace:GetDescendants()) do
        if v:IsA("PointLight") or v:IsA("SpotLight") or v:IsA("SurfaceLight") then
            if not originalLightBrightness[v] then
                originalLightBrightness[v] = v.Brightness
            end
            v.Brightness = originalLightBrightness[v] * 0.5
        end
    end
end

local function quality2023On()
    ajustarLuzes2023()

    table.insert(quality2023Connections, player.CharacterAdded:Connect(function()
        task.wait(0.5)
        if states["2023 QUALITY"] then
            ajustarLuzes2023()
        end
    end))

    table.insert(quality2023Connections, Workspace.DescendantAdded:Connect(function(v)
        if states["2023 QUALITY"] and (v:IsA("PointLight") or v:IsA("SpotLight") or v:IsA("SurfaceLight")) then
            task.wait()
            if not originalLightBrightness[v] then
                originalLightBrightness[v] = v.Brightness
            end
            v.Brightness = originalLightBrightness[v] * 0.5
        end
    end))

    print("✅ 2023 QUALITY ON")
end

local function quality2023Off()
    for _, conn in ipairs(quality2023Connections) do
        pcall(function() conn:Disconnect() end)
    end
    quality2023Connections = {}

    for light, origBrightness in pairs(originalLightBrightness) do
        if light and light.Parent then
            light.Brightness = origBrightness
        end
    end
    originalLightBrightness = {}

    print("❌ 2023 QUALITY OFF")
end

--========================================================
-- 2020 QUALITY
--========================================================

local qualityConnections = {}
local origLightingSettings = {}
local disabledLightsList = {}

local function quality2020On()
    origLightingSettings = {
        Technology = Lighting.Technology,
        GlobalShadows = Lighting.GlobalShadows,
        Brightness = Lighting.Brightness,
        ClockTime = Lighting.ClockTime,
        ExposureCompensation = Lighting.ExposureCompensation,
        Ambient = Lighting.Ambient,
        OutdoorAmbient = Lighting.OutdoorAmbient,
        FogStart = Lighting.FogStart,
        FogEnd = Lighting.FogEnd,
        FogColor = Lighting.FogColor,
        EnvironmentDiffuseScale = Lighting.EnvironmentDiffuseScale,
        EnvironmentSpecularScale = Lighting.EnvironmentSpecularScale,
        ColorShift_Top = Lighting.ColorShift_Top,
        ColorShift_Bottom = Lighting.ColorShift_Bottom
    }

    for _, v in ipairs(Lighting:GetChildren()) do
        if v:IsA("Atmosphere") or v:IsA("BloomEffect") or v:IsA("SunRaysEffect") or v:IsA("ColorCorrectionEffect") or v:IsA("DepthOfFieldEffect") then
            v:Destroy()
        end
    end

    pcall(function() Lighting.Technology = Enum.Technology.Compatibility end)
    Lighting.GlobalShadows = false
    Lighting.Brightness = 2.12
    Lighting.ClockTime = 14
    Lighting.ExposureCompensation = 0
    Lighting.Ambient = Color3.fromRGB(148, 148, 148)
    Lighting.OutdoorAmbient = Color3.fromRGB(148, 148, 148)
    Lighting.FogStart = 100000
    Lighting.FogEnd = 100000
    Lighting.FogColor = Color3.fromRGB(255, 255, 255)
    Lighting.EnvironmentDiffuseScale = 0
    Lighting.EnvironmentSpecularScale = 0

    pcall(function()
        Lighting.ColorShift_Top = Color3.new(0, 0, 0)
        Lighting.ColorShift_Bottom = Color3.new(0, 0, 0)
    end)

    table.insert(qualityConnections, Lighting.ChildAdded:Connect(function(obj)
        if not states["2020 QUALITY"] then return end
        if obj:IsA("Atmosphere") or obj:IsA("BloomEffect") or obj:IsA("SunRaysEffect") or obj:IsA("ColorCorrectionEffect") or obj:IsA("DepthOfFieldEffect") then
            task.wait()
            if obj.Parent then obj:Destroy() end
        end
    end))

    local function disableLight(obj)
        if obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") then
            if obj.Enabled then
                disabledLightsList[obj] = true
                obj.Enabled = false
            end
        end
    end

    for _, obj in ipairs(Workspace:GetDescendants()) do
        disableLight(obj)
    end

    table.insert(qualityConnections, Workspace.DescendantAdded:Connect(function(obj)
        if states["2020 QUALITY"] then
            disableLight(obj)
        end
    end))

    print("✅ 2020 QUALITY ON")
end

local function quality2020Off()
    for _, conn in ipairs(qualityConnections) do
        pcall(function() conn:Disconnect() end)
    end
    qualityConnections = {}

    if origLightingSettings.Technology then
        pcall(function() Lighting.Technology = origLightingSettings.Technology end)
        Lighting.GlobalShadows = origLightingSettings.GlobalShadows
        Lighting.Brightness = origLightingSettings.Brightness
        Lighting.ClockTime = origLightingSettings.ClockTime
        Lighting.ExposureCompensation = origLightingSettings.ExposureCompensation
        Lighting.Ambient = origLightingSettings.Ambient
        Lighting.OutdoorAmbient = origLightingSettings.OutdoorAmbient
        Lighting.FogStart = origLightingSettings.FogStart
        Lighting.FogEnd = origLightingSettings.FogEnd
        Lighting.FogColor = origLightingSettings.FogColor
        Lighting.EnvironmentDiffuseScale = origLightingSettings.EnvironmentDiffuseScale
        Lighting.EnvironmentSpecularScale = origLightingSettings.EnvironmentSpecularScale
        pcall(function()
            Lighting.ColorShift_Top = origLightingSettings.ColorShift_Top
            Lighting.ColorShift_Bottom = origLightingSettings.ColorShift_Bottom
        end)
    end

    for light, _ in pairs(disabledLightsList) do
        if light and light.Parent then
            light.Enabled = true
        end
    end
    disabledLightsList = {}

    print("❌ 2020 QUALITY OFF")
end

--========================================================
-- TIRAR X DA GUN
--========================================================

local cleanGunConnections = {}

local function processGunChild(child)
    if not states["TIRAR X DA GUN"] then return end
    if child:IsA("BillboardGui") or child:IsA("SurfaceGui") then
        child.Enabled = false
        table.insert(cleanGunConnections, child:GetPropertyChangedSignal("Enabled"):Connect(function()
            if states["TIRAR X DA GUN"] then child.Enabled = false end
        end))
    elseif child:IsA("ImageLabel") or child:IsA("ImageButton") or child:IsA("Decal") then
        if child.Name:lower() == "x" or child.Name:lower():find("cross") or child.Name:lower():find("block") then
            child.Visible = false
            child.Transparency = 1
        end
    end
end

local function cleanWeaponX(tool)
    if not tool:IsA("Tool") or not states["TIRAR X DA GUN"] then return end

    for _, descendant in ipairs(tool:GetDescendants()) do
        processGunChild(descendant)
    end

    table.insert(cleanGunConnections, tool.DescendantAdded:Connect(processGunChild))
end

local function setupCharacterGun(char)
    if not char then return end

    table.insert(cleanGunConnections, char.ChildAdded:Connect(cleanWeaponX))
    for _, child in pairs(char:GetChildren()) do
        cleanWeaponX(child)
    end
end

local function tirarXGunOn()
    if player.Character then setupCharacterGun(player.Character) end
    table.insert(cleanGunConnections, player.CharacterAdded:Connect(setupCharacterGun))

    print("✅ TIRAR X DA GUN ON")
end

local function tirarXGunOff()
    for _, conn in ipairs(cleanGunConnections) do
        pcall(function() conn:Disconnect() end)
    end
    cleanGunConnections = {}

    print("❌ TIRAR X DA GUN OFF")
end

--========================================================
-- KORBLOX BIEL (R15 / R6)
--========================================================

local korbloxBielConnections = {}

local function applyKorbloxBiel()
    if not states["KORBLOX BIEL"] then return end
    local char = player.Character or player.CharacterAdded:Wait()
    local hum = char:WaitForChild("Humanoid", 5)

    if not hum or char:FindFirstChild("KorbloxFolder") then 
        return 
    end

    local folder = Instance.new("Folder")
    folder.Name = "KorbloxFolder"
    folder.Parent = char

    if hum.RigType == Enum.HumanoidRigType.R15 then
        local ru = char:WaitForChild("RightUpperLeg", 3)
        local rl = char:WaitForChild("RightLowerLeg", 3)
        local rf = char:WaitForChild("RightFoot", 3)

        if not (ru and rl and rf) then return end

        ru.Transparency = 1
        rl.Transparency = 1
        rf.Transparency = 1

        local korbloxPart = Instance.new("Part")
        korbloxPart.Name = "KorbloxR15Visual"
        korbloxPart.Size = Vector3.new(1, 2, 1)
        korbloxPart.CanCollide = false
        korbloxPart.Massless = true
        korbloxPart.Parent = folder

        local mesh = Instance.new("SpecialMesh")
        mesh.MeshId = "rbxassetid://902942096"
        mesh.TextureId = "rbxassetid://902843398"
        mesh.Scale = Vector3.new(1, 1, 1)
        mesh.Parent = korbloxPart

        local weld = Instance.new("Weld")
        weld.Part0 = ru
        weld.Part1 = korbloxPart
        weld.C0 = CFrame.new(0.1, 0.2, 0) 
        weld.Parent = korbloxPart

    elseif hum.RigType == Enum.HumanoidRigType.R6 then
        local rightLeg = char:WaitForChild("Right Leg", 3)
        if not rightLeg then return end

        for _, v in ipairs(char:GetChildren()) do
            if v:IsA("CharacterMesh") and v.BodyPart == Enum.BodyPart.RightLeg then
                v:Destroy()
            end
        end

        rightLeg.Transparency = 1

        local korbloxLeg = Instance.new("Part")
        korbloxLeg.Name = "KorbloxR6Visual"
        korbloxLeg.Size = Vector3.new(1, 2, 1)
        korbloxLeg.CanCollide = false
        korbloxLeg.Massless = true
        korbloxLeg.Parent = folder

        local mesh = Instance.new("SpecialMesh")
        mesh.MeshType = Enum.MeshType.FileMesh
        mesh.MeshId = "rbxassetid://101851696"
        mesh.TextureId = "rbxassetid://101851254"
        mesh.Scale = Vector3.new(1, 1, 1)
        mesh.Parent = korbloxLeg

        local weld = Instance.new("Weld")
        weld.Part0 = rightLeg
        weld.Part1 = korbloxLeg
        weld.C0 = CFrame.new(0, 0, 0)
        weld.Parent = korbloxLeg
    end
end

local function korbloxBielOn()
    if player.Character then
        task.spawn(applyKorbloxBiel)
    end

    table.insert(korbloxBielConnections, player.CharacterAdded:Connect(function()
        task.wait(0.5)
        applyKorbloxBiel()
    end))

    print("✅ KORBLOX BIEL ON")
end

local function korbloxBielOff()
    for _, conn in ipairs(korbloxBielConnections) do
        pcall(function() conn:Disconnect() end)
    end
    korbloxBielConnections = {}

    local char = player.Character
    if char then
        local folder = char:FindFirstChild("KorbloxFolder")
        if folder then folder:Destroy() end

        local hum = char:FindFirstChild("Humanoid")
        if hum then
            if hum.RigType == Enum.HumanoidRigType.R15 then
                for _, name in ipairs({"RightUpperLeg", "RightLowerLeg", "RightFoot"}) do
                    local part = char:FindFirstChild(name)
                    if part then part.Transparency = 0 end
                end
            elseif hum.RigType == Enum.HumanoidRigType.R6 then
                local rl = char:FindFirstChild("Right Leg")
                if rl then rl.Transparency = 0 end
            end
        end
    end

    print("❌ KORBLOX BIEL OFF")
end

--========================================================
-- WATERBALLOON
--========================================================

local waterballoonConnections = {}
local waterballoonThread
local WATERBALLOON_IMAGE_ID = "rbxassetid://89108586792149"
local THROW_ANIMATION_ID = "rbxassetid://18324269091"
local COOLDOWN_TIME = 2.5
local HIDE_DELAY = 0.6
local canThrowBalloon = true

local function createWaterballoonTool()
    if not states["WATERBALLOON"] then return end
    local backpack = player:WaitForChild("Backpack", 5)
    if not backpack then return end

    if backpack:FindFirstChild("waterballoon") or (player.Character and player.Character:FindFirstChild("waterballoon")) then
        return
    end

    tool = Instance.new("Tool")
    tool.Name = "waterballoon"
    tool.RequiresHandle = true
    tool.CanBeDropped = false
    tool.TextureId = WATERBALLOON_IMAGE_ID
    tool.Grip = CFrame.new(0.2, 0, 0) * CFrame.Angles(0, 0, math.rad(90))

    local handle = Instance.new("Part")
    handle.Name = "Handle"
    handle.Size = Vector3.new(1.2, 1.2, 1.2)
    handle.CanCollide = false
    handle.CanTouch = false
    handle.CanQuery = false
    handle.Massless = true
    handle.Color = Color3.fromRGB(235, 64, 88)
    handle.Material = Enum.Material.SmoothPlastic
    handle.Parent = tool

    local mesh = Instance.new("SpecialMesh")
    mesh.MeshId = "rbxassetid://137510221029430"
    mesh.Scale = Vector3.new(0.45, 0.45, 0.45)
    mesh.Parent = handle

    local throwTrack

    local function loadAnimation()
        local character = player.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return end
        local animator = humanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", humanoid)

        local animation = Instance.new("Animation")
        animation.AnimationId = THROW_ANIMATION_ID

        pcall(function()
            throwTrack = animator:LoadAnimation(animation)
        end)
    end

    loadAnimation()

    tool.Activated:Connect(function()
        if not canThrowBalloon or not tool.Parent then return end
        canThrowBalloon = false

        if not throwTrack then loadAnimation() end
        if throwTrack then pcall(function() throwTrack:Play() end) end

        task.wait(HIDE_DELAY)
        if handle and handle.Parent then handle.Transparency = 1 end

        task.wait(math.max(0, COOLDOWN_TIME - HIDE_DELAY))
        if handle and handle.Parent then handle.Transparency = 0 end

        canThrowBalloon = true
    end)

    tool.Parent = backpack
end

local function removeWaterballoonTool()
    local backpack = player:FindFirstChild("Backpack")
    if backpack then
        local item = backpack:FindFirstChild("waterballoon")
        if item then item:Destroy() end
    end
    if player.Character then
        local item = player.Character:FindFirstChild("waterballoon")
        if item then item:Destroy() end
    end
end

local function waterballoonOn()
    canThrowBalloon = true
    createWaterballoonTool()

    table.insert(waterballoonConnections, player.CharacterAdded:Connect(function()
        task.wait(1)
        canThrowBalloon = true
        createWaterballoonTool()
    end))

    waterballoonThread = task.spawn(function()
        while states["WATERBALLOON"] do
            task.wait(2)
            local backpack = player:FindFirstChild("Backpack")
            if backpack then
                local hasTool = backpack:FindFirstChild("waterballoon") or (player.Character and player.Character:FindFirstChild("waterballoon"))
                if not hasTool then createWaterballoonTool() end
            end
        end
    end)

    print("✅ WATERBALLOON ON")
end

local function waterballoonOff()
    for _, conn in ipairs(waterballoonConnections) do
        pcall(function() conn:Disconnect() end)
    end
    waterballoonConnections = {}

    if waterballoonThread then
        task.cancel(waterballoonThread)
        waterballoonThread = nil
    end

    removeWaterballoonTool()
    print("❌ WATERBALLOON OFF")
end

--========================================================
-- ANIM ANTIGA
--========================================================

local originalRetargeting

pcall(function()
    originalRetargeting = workspace.Retargeting
end)

local function animAntigaOn()
    pcall(function()
        workspace.Retargeting = Enum.AnimatorRetargetingMode.Disabled
    end)
    print("✅ ANIM ANTIGA ON")
end

local function animAntigaOff()
    pcall(function()
        if originalRetargeting then
            workspace.Retargeting = originalRetargeting
        end
    end)
    print("❌ ANIM ANTIGA OFF")
end

--========================================================
-- CONFETE
--========================================================

local confeteConnections = {}
local LAG_DURATION = 1
local REPLICATION_LAG = 0.6

local function simulateLag()
    if not states["CONFETE"] then return end
    local NetworkSettings = settings():GetService("NetworkSettings")

    pcall(function()
        NetworkSettings.IncomingReplicationLag = REPLICATION_LAG
        NetworkSettings.OutgoingReplicationLag = REPLICATION_LAG
    end)

    task.delay(LAG_DURATION, function()
        pcall(function()
            NetworkSettings.IncomingReplicationLag = 0
            NetworkSettings.OutgoingReplicationLag = 0
        end)
    end)
end

local function monitorConfete(character)
    local humanoid = character:FindFirstChildOfClass("Humanoid") or character:WaitForChild("Humanoid", 5)
    if not humanoid then return end

    table.insert(confeteConnections, humanoid.Died:Connect(simulateLag))
end

local function confeteOff()
    for _, connection in ipairs(confeteConnections) do
        pcall(function() connection:Disconnect() end)
    end
    confeteConnections = {}

    local NetworkSettings
    pcall(function()
        NetworkSettings = settings():GetService("NetworkSettings")
    end)
    if NetworkSettings then
        pcall(function()
            NetworkSettings.IncomingReplicationLag = 0
            NetworkSettings.OutgoingReplicationLag = 0
        end)
    end

    print("❌ CONFETE OFF")
end

local function confeteOn()
    confeteOff()
    states["CONFETE"] = true

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr.Character then monitorConfete(plr.Character) end
        table.insert(confeteConnections, plr.CharacterAdded:Connect(monitorConfete))
    end

    table.insert(confeteConnections, Players.PlayerAdded:Connect(function(plr)
        table.insert(confeteConnections, plr.CharacterAdded:Connect(monitorConfete))
    end))

    print("✅ CONFETE ON")
end

--========================================================
-- SOM DA ARMA
--========================================================

local weaponSound
local weaponConnections = {}
local SOUND_ID = "rbxassetid://6968135315"
local TARGET_TOOL_NAME = "Gun"
local COOLDOWN_DURATION = 5

local function weaponSoundOn()
    weaponSound = Instance.new("Sound")
    weaponSound.SoundId = SOUND_ID
    weaponSound.Volume = 1
    weaponSound.Parent = SoundService

    local function setupCharacter(character)
        local humanoid = character:WaitForChild("Humanoid", 10)
        local animator = humanoid and humanoid:WaitForChild("Animator", 10)
        if not animator then return end

        local lastUnequippedTime = 0

        table.insert(weaponConnections, animator.AnimationPlayed:Connect(function(track)
            if not states["SOM DA ARMA"] then return end
            local tool = character:FindFirstChildOfClass("Tool")
            local isHoldingGun = tool and tool.Name == TARGET_TOOL_NAME
            local timeSinceUnequip = tick() - lastUnequippedTime

            if (isHoldingGun or timeSinceUnequip <= COOLDOWN_DURATION) and track.Priority == Enum.AnimationPriority.Action then
                track:Stop()
            end
        end))

        table.insert(weaponConnections, character.ChildAdded:Connect(function(child)
            if child:IsA("Tool") and child.Name == TARGET_TOOL_NAME and weaponSound then
                weaponSound:Play()
            end
        end))

        table.insert(weaponConnections, character.ChildRemoved:Connect(function(child)
            if child:IsA("Tool") and child.Name == TARGET_TOOL_NAME then
                if weaponSound then weaponSound:Play() end
                lastUnequippedTime = tick()
            end
        end))
    end

    if player.Character then setupCharacter(player.Character) end
    table.insert(weaponConnections, player.CharacterAdded:Connect(setupCharacter))

    print("✅ SOM DA ARMA ON")
end

local function weaponSoundOff()
    for _, connection in ipairs(weaponConnections) do
        pcall(function() connection:Disconnect() end)
    end
    weaponConnections = {}

    if weaponSound then
        weaponSound:Destroy()
        weaponSound = nil
    end

    print("❌ SOM DA ARMA OFF")
end

--========================================================
-- SPEED
--========================================================

local activeGhost
local activeWeld
local currentOffset = CFrame.new()
local speedConnections = {}

local function isGlitchTarget(child)
    if child:IsA("Tool") then return true end
    return string.find(string.lower(child.Name), "radio") ~= nil
end

local function deployGhost()
    if not states["SPEED"] then return end
    local character = player.Character
    if not character then return end

    local root = character:FindFirstChild("HumanoidRootPart")
    local torso = character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso")
    if not root or not torso then return end

    local rawCF = root.CFrame:ToObjectSpace(torso.CFrame)
    currentOffset = CFrame.new(rawCF.Position * 1.9) * rawCF.Rotation

    if activeGhost then activeGhost:Destroy() end

    local ghost = Instance.new("Part")
    ghost.Name = "Skibidi_Ghost_Active"
    ghost.Size = Vector3.new(1, 1, 1)
    ghost.Transparency = 1
    ghost.CanCollide = false
    ghost.CustomPhysicalProperties = PhysicalProperties.new(20, 0.7, 0.3, 1, 1)
    ghost.CFrame = torso.CFrame
    ghost.Parent = character

    local weld = Instance.new("Weld")
    weld.Part0 = torso
    weld.Part1 = ghost
    weld.C0 = currentOffset
    weld.Parent = ghost

    activeGhost = ghost
    activeWeld = weld
end

local function speedOn()
    if states["SPEED RAW"] then states["SPEED RAW"] = false; if offFunctions and offFunctions["SPEED RAW"] then offFunctions["SPEED RAW"]() end; updateButton("SPEED RAW") end
    if states["SPEED CWLLES"] then states["SPEED CWLLES"] = false; if offFunctions and offFunctions["SPEED CWLLES"] then offFunctions["SPEED CWLLES"]() end; updateButton("SPEED CWLLES") end

    local function setupCharacter(character)
        task.defer(function()
            if states["SPEED"] then deployGhost() end
        end)

        table.insert(speedConnections, character.ChildAdded:Connect(function(child)
            if not states["SPEED"] then return end
            if isGlitchTarget(child) then
                for _, part in ipairs(child:GetDescendants()) do
                    if part:IsA("BasePart") then part.Massless = true end
                end
                deployGhost()
            end
        end))

        table.insert(speedConnections, character.ChildRemoved:Connect(function(child)
            if states["SPEED"] and isGlitchTarget(child) then deployGhost() end
        end))
    end

    if player.Character then setupCharacter(player.Character) end
    table.insert(speedConnections, player.CharacterAdded:Connect(setupCharacter))

    table.insert(speedConnections, RunService.Heartbeat:Connect(function()
        if not states["SPEED"] then return end
        if activeWeld then
            activeWeld.C0 = currentOffset
        elseif player.Character then
            deployGhost()
        end
    end))

    print("✅ SPEED ON")
end

local function speedOff()
    for _, connection in ipairs(speedConnections) do
        pcall(function() connection:Disconnect() end)
    end
    speedConnections = {}

    if activeGhost then pcall(function() activeGhost:Destroy() end) end
    activeGhost = nil
    activeWeld = nil
    currentOffset = CFrame.new()

    print("❌ SPEED OFF")
end

--========================================================
-- CORPO ANTIGO
--========================================================

local bodyConnections = {}
local bodySession = 0

local function applyBody(character, session)
    if not character or session ~= bodySession or not states["CORPO ANTIGO"] then return end
    local humanoid = character:WaitForChild("Humanoid", 10)
    if session ~= bodySession or not states["CORPO ANTIGO"] or not humanoid or humanoid.RigType ~= Enum.HumanoidRigType.R15 then return end

    humanoid.AutomaticScalingEnabled = false

    local height = humanoid:FindFirstChild("BodyHeightScale")
    local width = humanoid:FindFirstChild("BodyWidthScale")
    local depth = humanoid:FindFirstChild("BodyDepthScale")
    local head = humanoid:FindFirstChild("HeadScale")

    if height then height.Value = 1 end
    if width then width.Value = 1 end
    if depth then depth.Value = 0.8 end
    if head then head.Value = 1 end

    task.wait(0.2)

    if session ~= bodySession or not states["CORPO ANTIGO"] then
        humanoid.AutomaticScalingEnabled = true
        if height then height.Value = 1 end
        if width then width.Value = 1 end
        if depth then depth.Value = 1 end
        if head then head.Value = 1 end
        return
    end

    humanoid.AutomaticScalingEnabled = true
end

local function bodyOn()
    bodySession += 1
    local session = bodySession

    for _, connection in ipairs(bodyConnections) do
        pcall(function() connection:Disconnect() end)
    end
    bodyConnections = {}

    if player.Character then
        task.spawn(function() applyBody(player.Character, session) end)
    end

    table.insert(bodyConnections, player.CharacterAdded:Connect(function(character)
        if not states["CORPO ANTIGO"] then return end
        local currentSession = bodySession
        task.spawn(function() applyBody(character, currentSession) end)
    end))

    print("✅ CORPO ANTIGO ON")
end

local function bodyOff()
    bodySession += 1
    states["CORPO ANTIGO"] = false

    for _, connection in ipairs(bodyConnections) do
        pcall(function() connection:Disconnect() end)
    end
    bodyConnections = {}

    local character = player.Character
    if character then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.AutomaticScalingEnabled = true
            local height = humanoid:FindFirstChild("BodyHeightScale")
            local width = humanoid:FindFirstChild("BodyWidthScale")
            local depth = humanoid:FindFirstChild("BodyDepthScale")
            local head = humanoid:FindFirstChild("HeadScale")

            if height then height.Value = 1 end
            if width then width.Value = 1 end
            if depth then depth.Value = 1 end
            if head then head.Value = 1 end
        end
    end

    print("❌ CORPO ANTIGO OFF")
end

--========================================================
-- 2023 XP
--========================================================

local xpConnections = {}
local originalPositions = {}

local function configurarXP(obj)
    if not obj:IsA("TextLabel") then return end

    local function atualizar()
        if not states["2023 XP"] then return end

        if string.find(obj.Text:lower(), "xp") then
            local pai = obj.Parent
            if pai and pai:IsA("GuiObject") and not originalPositions[pai] then
                originalPositions[pai] = pai.Position
                pai.Position = pai.Position + UDim2.new(0, 0, 0, 35)
            end
        end

        if string.match(obj.Text, "%dm%s*%d+s") and not originalPositions[obj] then
            originalPositions[obj] = obj.Position
            obj.Position = obj.Position + UDim2.new(0, 0, 0, 35)
        end
    end

    table.insert(xpConnections, obj:GetPropertyChangedSignal("Text"):Connect(atualizar))
    atualizar()
end

local function xp2023On()
    for _, v in ipairs(player.PlayerGui:GetDescendants()) do
        configurarXP(v)
    end

    table.insert(xpConnections, player.PlayerGui.DescendantAdded:Connect(configurarXP))
    print("✅ 2023 XP ON")
end

local function xp2023Off()
    for _, connection in ipairs(xpConnections) do
        pcall(function() connection:Disconnect() end)
    end
    xpConnections = {}

    for element, pos in pairs(originalPositions) do
        if element and element.Parent then
            element.Position = pos
        end
    end
    originalPositions = {}

    print("❌ 2023 XP OFF")
end

--========================================================
-- HOT BAR 2025
--========================================================

local hotbarConnections = {}

local function createTool()
    if not states["HOT BAR 2025"] then return end
    local backpack = player:WaitForChild("Backpack", 5)
    if not backpack then return end

    if backpack:FindFirstChild("Emotes") or (player.Character and player.Character:FindFirstChild("Emotes")) then
        return
    end

    local tool = Instance.new("Tool")
    tool.RequiresHandle = false
    tool.CanBeDropped = false

    local lang = string.lower(player.LocaleId or "en-us")
    local names = {
        ["pt-br"] = "Emotes",
        ["en-us"] = "Emotes",
        ["es-es"] = "Emotes",
        ["fr-fr"] = "Émotes",
        ["de-de"] = "Emotes"
    }

    tool.Name = names[lang] or "Emotes"
    tool.Parent = backpack

    task.wait()

    local tools = {}
    for _, item in ipairs(backpack:GetChildren()) do
        if item:IsA("Tool") then
            table.insert(tools, item)
        end
    end

    for i, t in ipairs(tools) do
        if t == tool then
            table.remove(tools, i)
            break
        end
    end

    table.insert(tools, 2, tool)

    for _, t in ipairs(tools) do t.Parent = nil end
    for _, t in ipairs(tools) do t.Parent = backpack end
end

local function removeTool()
    local backpack = player:FindFirstChild("Backpack")
    if backpack then
        local item = backpack:FindFirstChild("Emotes") or backpack:FindFirstChild("Émotes")
        if item then item:Destroy() end
    end
    if player.Character then
        local item = player.Character:FindFirstChild("Emotes") or player.Character:FindFirstChild("Émotes")
        if item then item:Destroy() end
    end
end

local function hotBar2025On()
    createTool()

    table.insert(hotbarConnections, player.CharacterAdded:Connect(function()
        task.wait(0.5)
        createTool()
    end))

    print("✅ HOT BAR 2025 ON")
end

local function hotBar2025Off()
    for _, connection in ipairs(hotbarConnections) do
        pcall(function() connection:Disconnect() end)
    end
    hotbarConnections = {}

    removeTool()
    print("❌ HOT BAR 2025 OFF")
end

--========================================================
-- 2023 SHIFT
--========================================================

local shift2023Connections = {}

local function applyToHumanoid2023(hum)
    if hum:IsA("Humanoid") then
        hum.AutomaticScalingEnabled = false
    end
end

local function scanCharacter2023(character)
    if not character then return end
    for _, obj in pairs(character:GetDescendants()) do
        applyToHumanoid2023(obj)
    end

    table.insert(shift2023Connections, character.DescendantAdded:Connect(function(obj)
        if states["2023 SHIFT"] then
            applyToHumanoid2023(obj)
        end
    end))
end

local function shift2023On()
    if player.Character then
        scanCharacter2023(player.Character)
    end

    table.insert(shift2023Connections, player.CharacterAdded:Connect(function(character)
        if states["2023 SHIFT"] then
            scanCharacter2023(character)
        end
    end))

    print("✅ 2023 SHIFT ON")
end

local function shift2023Off()
    for _, connection in ipairs(shift2023Connections) do
        pcall(function() connection:Disconnect() end)
    end
    shift2023Connections = {}

    if player.Character then
        local hum = player.Character:FindFirstChildOfClass("Humanoid")
        if hum then
            hum.AutomaticScalingEnabled = true
        end
    end

    print("❌ 2023 SHIFT OFF")
end

--========================================================
-- 2024 SHIFT
--========================================================

local shift2024Connections = {}
local shiftOffset2024 = Vector3.new(1.2, 0.8, 0)
local normalOffset2024 = Vector3.new(0, 0, 0)
local humanoid2024
local isShiftLocked2024 = false

local function shift2024Off()
    for _, connection in ipairs(shift2024Connections) do
        pcall(function() connection:Disconnect() end)
    end
    shift2024Connections = {}

    if humanoid2024 and humanoid2024.Parent then
        humanoid2024.CameraOffset = normalOffset2024
    end

    humanoid2024 = nil
    isShiftLocked2024 = false
    print("❌ 2024 SHIFT OFF")
end

local function shift2024On()
    shift2024Off()

    local function onCharacterAdded(character)
        humanoid2024 = character:WaitForChild("Humanoid", 10)
        if humanoid2024 then
            humanoid2024.CameraOffset = normalOffset2024
            isShiftLocked2024 = false
        end
    end

    table.insert(shift2024Connections, player.CharacterAdded:Connect(onCharacterAdded))

    if player.Character then
        onCharacterAdded(player.Character)
    end

    table.insert(shift2024Connections, RunService.RenderStepped:Connect(function()
        if not states["2024 SHIFT"] or not humanoid2024 then return end

        if UserInputService.MouseBehavior == Enum.MouseBehavior.LockCenter then
            if not isShiftLocked2024 then
                isShiftLocked2024 = true
                humanoid2024.CameraOffset = shiftOffset2024
            end
        else
            if isShiftLocked2024 then
                isShiftLocked2024 = false
                humanoid2024.CameraOffset = normalOffset2024
            end
        end
    end))

    print("✅ 2024 SHIFT ON")
end

--========================================================
-- HOT BAR TEXT
--========================================================

local hotbarTextConnections = {}
local cachedLabelsText = {}
local SMALL_TEXT_SIZE_HOTBAR = 12
local TARGET_POSITION_HOTBAR = UDim2.new(0, -1, 0, -4)

local function isHotbarSlotLabel(label)
    if not (label and label:IsA("TextLabel") and label.Parent) then return false end

    local rawText = label.Text:gsub("[%s%.]+", "")
    local num = tonumber(rawText)
    if not num or num < 0 or num > 10 then return false end

    local current = label.Parent
    local isHotbar = false

    while current and current ~= game do
        local name = current.Name:lower()

        if name:find("shop") or name:find("trade") or name:find("profile") or 
           name:find("leaderboard") or name:find("crafting") or name:find("level") or 
           name:find("coin") or name:find("menu") or name:find("hud") then
            return false
        end

        if name:find("hotbar") or name:find("backpack") or name:find("toolbar") or name:find("slot") then
            isHotbar = true
        end

        current = current.Parent
    end

    return isHotbar
end

local function checkAndCacheText(descendant)
    if descendant:IsA("TextLabel") and isHotbarSlotLabel(descendant) then
        cachedLabelsText[descendant] = true
    end
end

local function scanAndCacheText(container)
    if not container then return end
    for _, obj in ipairs(container:GetDescendants()) do
        checkAndCacheText(obj)
    end
end

local function hotBarTextOn()
    if _G.TinyHotbarThread then
        task.cancel(_G.TinyHotbarThread)
        _G.TinyHotbarThread = nil
    end

    local pGui = player:FindFirstChild("PlayerGui")
    if pGui then 
        scanAndCacheText(pGui)
        table.insert(hotbarTextConnections, pGui.DescendantAdded:Connect(checkAndCacheText))
    end

    pcall(function()
        local coreGui = game:GetService("CoreGui")
        scanAndCacheText(coreGui)
        table.insert(hotbarTextConnections, coreGui.DescendantAdded:Connect(checkAndCacheText))
    end)

    table.insert(hotbarTextConnections, player.CharacterAdded:Connect(function()
        task.wait(0.5)
        table.clear(cachedLabelsText)
        local currentPGui = player:FindFirstChild("PlayerGui")
        if currentPGui then scanAndCacheText(currentPGui) end
        pcall(function() scanAndCacheText(game:GetService("CoreGui")) end)
    end))

    _G.TinyHotbarThread = task.spawn(function()
        while states["HOT BAR TEXT"] do
            for label in pairs(cachedLabelsText) do
                if label and label.Parent then
                    if label.TextScaled then label.TextScaled = false end
                    if label.TextSize ~= SMALL_TEXT_SIZE_HOTBAR then label.TextSize = SMALL_TEXT_SIZE_HOTBAR end
                    if label.AnchorPoint ~= Vector2.new(0, 0) then label.AnchorPoint = Vector2.new(0, 0) end
                    if label.Position ~= TARGET_POSITION_HOTBAR then label.Position = TARGET_POSITION_HOTBAR end

                    label.Size = UDim2.new(0, 14, 0, 14)
                    label.TextXAlignment = Enum.TextXAlignment.Left
                    label.TextYAlignment = Enum.TextYAlignment.Top
                else
                    cachedLabelsText[label] = nil
                end
            end
            task.wait(0.4)
        end
    end)

    print("✅ HOT BAR TEXT ON")
end

local function hotBarTextOff()
    for _, connection in ipairs(hotbarTextConnections) do
        pcall(function() connection:Disconnect() end)
    end
    hotbarTextConnections = {}

    if _G.TinyHotbarThread then
        task.cancel(_G.TinyHotbarThread)
        _G.TinyHotbarThread = nil
    end

    table.clear(cachedLabelsText)
    print("❌ HOT BAR TEXT OFF")
end

--========================================================
-- VOICE 2023
--========================================================
local voice2023Connections = {}
local RAW_ASSET_ID = "97664450574335"
local IMAGE_URI = "rbxthumb://type=Asset&id=" .. RAW_ASSET_ID .. "&w=420&h=420"
local function voice2023Create(char)
    if not states["VOICE 2023"] then return end
    local head = char:WaitForChild("Head", 5)
    if not head then return end
    local existing = head:FindFirstChild("CustomVoiceBubble")
    if existing then existing:Destroy() end
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "CustomVoiceBubble"; billboard.Size = UDim2.new(0,30,0,30); billboard.StudsOffset = Vector3.new(0,2.2,0); billboard.AlwaysOnTop=true; billboard.Active=true; billboard.Adornee=head; billboard.Parent=head
    local button=Instance.new("TextButton"); button.Name="BubbleButton"; button.Size=UDim2.new(1,0,1,0); button.BackgroundColor3=Color3.fromRGB(240,240,240); button.BorderSizePixel=0; button.Text=""; button.AutoButtonColor=true; button.Active=true; button.Selectable=true; button.ClipsDescendants=false; button.Parent=billboard
    local c=Instance.new("UICorner"); c.CornerRadius=UDim.new(0,8); c.Parent=button
    local tailHolder=Instance.new("Frame"); tailHolder.Name="TailHolder"; tailHolder.Size=UDim2.new(0,12,0,8); tailHolder.Position=UDim2.new(0.5,-6,1,-1); tailHolder.BackgroundTransparency=1; tailHolder.ClipsDescendants=true; tailHolder.Parent=button
    local rotatedSquare=Instance.new("Frame"); rotatedSquare.Name="PyramidTail"; rotatedSquare.Size=UDim2.new(0,8,0,8); rotatedSquare.Position=UDim2.new(0.5,-4,0,-4); rotatedSquare.Rotation=45; rotatedSquare.BackgroundColor3=Color3.fromRGB(240,240,240); rotatedSquare.BorderSizePixel=0; rotatedSquare.Parent=tailHolder
    local micIcon=Instance.new("ImageLabel"); micIcon.Name="MicIcon"; micIcon.Size=UDim2.new(0,18,0,18); micIcon.Position=UDim2.new(0.5,0,0.5,0); micIcon.AnchorPoint=Vector2.new(0.5,0.5); micIcon.BackgroundTransparency=1; micIcon.Image=IMAGE_URI; micIcon.ScaleType=Enum.ScaleType.Fit; micIcon.Parent=button
    local redSlash=Instance.new("Frame"); redSlash.Name="RedMuteSlash"; redSlash.Size=UDim2.new(0,18,0,2.5); redSlash.Position=UDim2.new(0.5,0,0.5,0); redSlash.AnchorPoint=Vector2.new(0.5,0.5); redSlash.Rotation=-45; redSlash.BackgroundColor3=Color3.fromRGB(215,65,65); redSlash.BorderSizePixel=0; redSlash.Visible=true; redSlash.Parent=button
    local lc=Instance.new("UICorner"); lc.CornerRadius=UDim.new(1,0); lc.Parent=redSlash
    local isMuted=true
    local function toggleVisualState() isMuted=not isMuted; redSlash.Visible=isMuted; micIcon.ImageColor3=isMuted and Color3.fromRGB(255,255,255) or Color3.fromRGB(45,180,105) end
    table.insert(voice2023Connections, button.MouseButton1Click:Connect(toggleVisualState))
    table.insert(voice2023Connections, button.TouchTap:Connect(toggleVisualState))
end
local function voice2023On()
    if player.Character then task.spawn(voice2023Create, player.Character) end
    table.insert(voice2023Connections, player.CharacterAdded:Connect(voice2023Create))
    print("✅ VOICE 2023 ON")
end
local function voice2023Off()
    for _,c in ipairs(voice2023Connections) do pcall(function() c:Disconnect() end) end; voice2023Connections={}
    if player.Character then local h=player.Character:FindFirstChild("Head"); if h then local b=h:FindFirstChild("CustomVoiceBubble"); if b then b:Destroy() end end end
    print("❌ VOICE 2023 OFF")
end

--========================================================
-- SNOWBALL
--========================================================
local function snowballOn()
    local ok, err = pcall(function()
        local snowballChunk, snowballCompileErr = ExecutorCompat.compile([==[
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Debris = game:GetService("Debris")
local UserInputService = game:GetService("UserInputService")
local VirtualInputManager
pcall(function() VirtualInputManager = game:GetService("VirtualInputManager") end)
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = Workspace.CurrentCamera
local TOOL_NAME = "SnowballToy"
local MIN_FORWARD_SPEED = 20
local MAX_FORWARD_SPEED = 120
local MIN_UPWARD_LIFT = 25
local MAX_UPWARD_LIFT = 25
local WINDUP_DELAY = 0.7
local RELOAD_TIME = 2.0
local COOLDOWN_TIME = 0.15

-- Runtime único: evita acumular conexões/slots quando desativa e ativa várias vezes.
if _G.N4T_SnowballCleanup then
    pcall(_G.N4T_SnowballCleanup)
end

local runtimeConnections = {}
local tool

local function bind(connection)
    table.insert(runtimeConnections, connection)
    return connection
end

local function removeSnowballUI()
    local pGui = LocalPlayer:FindFirstChild("PlayerGui")
    if not pGui then return end

    -- Primeiro remove linhas criadas exclusivamente para a Snowball.
    for _, obj in ipairs(pGui:GetDescendants()) do
        if obj.Name == "SnowballOnlyRow" or obj:GetAttribute("N4TSnowballOnlyRow") == true then
            pcall(function() obj:Destroy() end)
        end
    end

    -- Depois remove qualquer slot antigo restante, inclusive de versões anteriores.
    for _, obj in ipairs(pGui:GetDescendants()) do
        if obj.Name == "SnowballSlot" or obj:GetAttribute("N4TSnowballSlot") == true then
            pcall(function() obj:Destroy() end)
        end
    end
end

local function removeSnowballWorldObjects()
    if LocalPlayer.Backpack then
        for _, item in ipairs(LocalPlayer.Backpack:GetChildren()) do
            if item.Name == TOOL_NAME or item.Name == "SnowBallToy2020" then
                pcall(function() item:Destroy() end)
            end
        end
    end
    if LocalPlayer.Character then
        for _, item in ipairs(LocalPlayer.Character:GetChildren()) do
            if item.Name == TOOL_NAME or item.Name == "SnowBallToy2020" then
                pcall(function() item:Destroy() end)
            end
        end
    end
    for _, obj in ipairs(Workspace:GetChildren()) do
        if obj.Name == "SnowballProjectile" or obj.Name == "SnowballImpact" then
            pcall(function() obj:Destroy() end)
        end
    end
end

local function cleanupSnowballRuntime()
    for _, connection in ipairs(runtimeConnections) do
        pcall(function()
            if connection and connection.Connected then
                connection:Disconnect()
            end
        end)
    end
    table.clear(runtimeConnections)

    if tool then
        pcall(function() tool:Destroy() end)
    end

    removeSnowballWorldObjects()
    removeSnowballUI()

    if _G.N4T_SnowballCleanup == cleanupSnowballRuntime then
        _G.N4T_SnowballCleanup = nil
    end
end

_G.N4T_SnowballCleanup = cleanupSnowballRuntime
removeSnowballUI()

-- 1. PURGE OLD INSTANCES & UI ELEMENTS*
if LocalPlayer.Backpack then
    for _, item in ipairs(LocalPlayer.Backpack:GetChildren()) do
        if item.Name == TOOL_NAME or item.Name == "SnowBallToy2020" then 
            item:Destroy() 
        end
    end
end
if LocalPlayer.Character then
    for _, item in ipairs(LocalPlayer.Character:GetChildren()) do
        if item.Name == TOOL_NAME or item.Name == "SnowBallToy2020" then 
            item:Destroy() 
        end
    end
end
for _, obj in ipairs(Workspace:GetChildren()) do
    if obj.Name == "SnowballProjectile" or obj.Name == "SnowballImpact" then
        obj:Destroy()
    end
end
-- Helper to safely locate the Game Emotes UI Container*
local function getGameEmotesPage()
    local pGui = LocalPlayer:FindFirstChild("PlayerGui")
    if not pGui then return nil end

    -- Caminho real do menu T do MM2. Não procurar qualquer objeto chamado
    -- "Game Emotes", porque existe também a aba/botão com esse nome.
    local crossPlatform = pGui:FindFirstChild("CrossPlatform")
    if not crossPlatform then return nil end

    local emotes = crossPlatform:FindFirstChild("Emotes") or crossPlatform:FindFirstChild("Emotes", true)
    if not emotes then return nil end

    local emoteWindow = emotes:FindFirstChild("EmoteWindow", true)
    if not emoteWindow then return nil end

    local emoteContainer = emoteWindow:FindFirstChild("EmoteContainer", true)
    if not emoteContainer then return nil end

    local emotePages = emoteContainer:FindFirstChild("EmotePages", true)
    if not emotePages then return nil end

    local gameEmotes = emotePages:FindFirstChild("Game Emotes")
    if gameEmotes and (gameEmotes:IsA("GuiObject") or gameEmotes:IsA("Folder")) then
        return gameEmotes
    end
    return nil
end
-- Purge existing custom Emote UI rows to avoid duplicates when re-executing*
local gameEmotesPage = getGameEmotesPage()
if gameEmotesPage then
    for _, child in ipairs(gameEmotesPage:GetChildren()) do
        if child.Name == "SnowballRow" or child:FindFirstChild("SnowballSlot") then
            child:Destroy()
        end
    end
end
-- 2. CREATE TOOL CONTAINER & HANDLE*
local tool = Instance.new("Tool")
tool.Name = TOOL_NAME
tool.GripPos = Vector3.new(0.03, -0.24, -0.01)
tool.RequiresHandle = true
-- Hotbar Icon*
tool.TextureId = "rbxthumb://type=Asset&w=150&h=150&id=6123754554"
local handle = Instance.new("Part")
handle.Name = "Handle"
handle.Shape = Enum.PartType.Ball
handle.Size = Vector3.new(1, 1, 1)
handle.Material = Enum.Material.Sand
handle.Color = Color3.new(0.956863, 0.968627, 1)
handle.CanCollide = false
handle.Parent = tool

-- A Tool fica fora da hotbar até o jogador escolher Snowball no menu T do MM2.

-- 3. PARTICLES SETUP (Impact Burst Emitters)*
local p1 = Instance.new("ParticleEmitter")
p1.Name = "Effect1"
p1.Texture = "rbxassetid://2346141109"
p1.Enabled = false
p1.Rate = 20.0
p1.Speed = NumberRange.new(4.0, 4.0)
p1.Size = NumberSequence.new({
    NumberSequenceKeypoint.new(0, 0.1875),
    NumberSequenceKeypoint.new(0.726751, 0),
    NumberSequenceKeypoint.new(1, 0)
})
p1.Parent = handle
local p2 = Instance.new("ParticleEmitter")
p2.Name = "Effect2"
p2.Texture = "rbxassetid://1539332776"
p2.Enabled = false
p2.Rate = 10.0
p2.Speed = NumberRange.new(1.0, 1.0)
p2.Size = NumberSequence.new({
    NumberSequenceKeypoint.new(0, 2.0),
    NumberSequenceKeypoint.new(0.716258, 0.560748),
    NumberSequenceKeypoint.new(1, 0)
})
p2.Parent = handle
-- 4. ANIMATION & THROW LOGIC*
local throwAnim = Instance.new("Animation")
throwAnim.AnimationId = "rbxassetid://18324269091"
local debounce = false
local function isShiftlockActive()
    if UserInputService.MouseBehavior == Enum.MouseBehavior.LockCenter then
        return true
    end
    local char = LocalPlayer.Character
    if char then
        local root = char:FindFirstChild("HumanoidRootPart")
        if root and Camera then
            local camLook = Vector3.new(Camera.CFrame.LookVector.X, 0, Camera.CFrame.LookVector.Z).Unit
            local rootLook = Vector3.new(root.CFrame.LookVector.X, 0, root.CFrame.LookVector.Z).Unit
            if camLook:Dot(rootLook) > 0.95 and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
                return true
            end
        end
    end
    return false
end
local function getTargetPosition()
    if UserInputService.TouchEnabled and not UserInputService.MouseEnabled then
        local camera = Workspace.CurrentCamera
        if camera then
            local viewportCenter = camera.ViewportSize / 2
            local unitRay = camera:ViewportPointToRay(viewportCenter.X, viewportCenter.Y)
            return unitRay.Origin + (unitRay.Direction * 1000)
        end
    end
    return Mouse.Hit.Position
end
bind(tool.Activated:Connect(function()
    if debounce then return end
    debounce = true
    local char = LocalPlayer.Character
    if not char then 
        debounce = false 
        return 
    end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    if not hum or hum.Health <= 0 or not root then
        debounce = false
        return
    end
    local animator = hum:FindFirstChildOfClass("Animator") or hum
    local track = animator:LoadAnimation(throwAnim)
    track.Priority = Enum.AnimationPriority.Action4 
    track:Play()
    task.wait(WINDUP_DELAY)
    if handle then
        handle.Transparency = 1
    end
    if char and char.Parent and hum and hum.Health > 0 and root and root.Parent then
        local targetPos = getTargetPosition()
        local aimOrigin = root.Position + Vector3.new(0, 1.5, 0)
        local rawDir = targetPos - aimOrigin
        local distance = rawDir.Magnitude
        local forwardSpeed, upwardLift
        if isShiftlockActive() then
            forwardSpeed = MAX_FORWARD_SPEED
            upwardLift = MAX_UPWARD_LIFT
        elseif tool.Parent == char then
            forwardSpeed = math.clamp(distance * 2.2, MIN_FORWARD_SPEED, MAX_FORWARD_SPEED)
            upwardLift = math.clamp(distance * 0.25, MIN_UPWARD_LIFT, MAX_UPWARD_LIFT)
        else
            forwardSpeed = MAX_FORWARD_SPEED
            upwardLift = MAX_UPWARD_LIFT
        end
        local flatDir = Vector3.new(rawDir.X, 0, rawDir.Z)
        if flatDir.Magnitude > 0.1 then
            flatDir = flatDir.Unit
        else
            flatDir = Vector3.new(root.CFrame.LookVector.X, 0, root.CFrame.LookVector.Z).Unit
        end
        local rightHand = char:FindFirstChild("RightHand") or char:FindFirstChild("Right Arm")
        local spawnPos
        if tool.Parent == char and handle then
            spawnPos = handle.Position
        elseif rightHand then
            spawnPos = rightHand.Position
        else
            spawnPos = aimOrigin
        end
        local projectile = Instance.new("Part")
        projectile.Name = "SnowballProjectile"
        projectile.Shape = Enum.PartType.Ball
        projectile.Size = Vector3.new(1.00, 1.00, 1.00)
        projectile.Material = Enum.Material.Sand
        projectile.Color = Color3.new(0.956863, 0.968627, 1)
        projectile.CanCollide = false
        projectile.Anchored = false
        projectile.CFrame = CFrame.new(spawnPos, spawnPos + flatDir)
        projectile.Parent = Workspace
        projectile.AssemblyLinearVelocity = (flatDir * forwardSpeed) + Vector3.new(0, upwardLift, 0)
        local hitConn
        hitConn = bind(projectile.Touched:Connect(function(hit)
            if hit:IsDescendantOf(char) or hit.Name == "SnowballProjectile" then return end
            if hitConn then
                hitConn:Disconnect()
                hitConn = nil
            end
            local impact = Instance.new("Part")
            impact.Name = "SnowballImpact"
            impact.Size = Vector3.new(0.1, 0.1, 0.1)
            impact.Transparency = 1
            impact.Anchored = true
            impact.CanCollide = false
            impact.CFrame = projectile.CFrame
            impact.Parent = Workspace
            local b1 = p1:Clone()
            b1.SpreadAngle = Vector2.new(180, 180)
            b1.Lifetime = NumberRange.new(1.0, 1.0)
            b1.Parent = impact
            local b2 = p2:Clone()
            b2.SpreadAngle = Vector2.new(90, 90)
            b2.Lifetime = NumberRange.new(1.0, 1.0)
            b2.Parent = impact
            b1:Emit(10)
            b2:Emit(1)
            Debris:AddItem(impact, 1.0)
            projectile:Destroy()
        end))
        Debris:AddItem(projectile, 4)
        task.delay(4, function()
            if hitConn then
                hitConn:Disconnect()
                hitConn = nil
            end
        end)
    end
    task.wait(RELOAD_TIME)
    if handle then
        handle.Transparency = 0
    end
    task.wait(COOLDOWN_TIME)
    debounce = false
end))
-- 5. UI INJECTION & SMART EMOTE MENU KEYBIND*
local function isEmotesMenuOpen()
    local pGui = LocalPlayer:FindFirstChild("PlayerGui")
    if pGui then
        local crossPlatform = pGui:FindFirstChild("CrossPlatform")
        if crossPlatform then
            local emoteWindow = crossPlatform:FindFirstChild("EmoteWindow", true)
            if emoteWindow and emoteWindow:IsA("GuiObject") and emoteWindow.Visible then
                return true
            end
        end
    end
    return false
end
local function closeEmotesMenu()
    -- Try simulating keypress first (PC)*
    if VirtualInputManager then
        pcall(function()
            VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.T, false, game)
            task.wait(0.03)
            VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.T, false, game)
        end)
    end
    -- Mobile / Backup fallback: Directly hide EmoteWindow if it's still open*
    task.delay(0.05, function()
        local pGui = LocalPlayer:FindFirstChild("PlayerGui")
        if pGui then
            local crossPlatform = pGui:FindFirstChild("CrossPlatform")
            if crossPlatform then
                local emoteWindow = crossPlatform:FindFirstChild("EmoteWindow", true)
                if emoteWindow and emoteWindow:IsA("GuiObject") then
                    emoteWindow.Visible = false
                end
            end
        end
    end)
end
local function equipSnowballAndCloseMenu()
    if not LocalPlayer.Backpack then return end
    if tool.Parent ~= LocalPlayer.Backpack and tool.Parent ~= LocalPlayer.Character then
        tool.Parent = LocalPlayer.Backpack
    end
    if LocalPlayer.Character then
        local hum = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then
            hum:EquipTool(tool)
        end
    end
    closeEmotesMenu()
end
-- O número mostrado/atalho acompanha a posição real da Snowball na linha.
local snowballSlotNumber = 1
local snowballHotkey = Enum.KeyCode.One
local snowballKeypadHotkey = Enum.KeyCode.KeypadOne

local function setSnowballHotkey(slotNumber)
    snowballSlotNumber = math.clamp(tonumber(slotNumber) or 1, 1, 4)
    local normalKeys = {
        [1] = Enum.KeyCode.One,
        [2] = Enum.KeyCode.Two,
        [3] = Enum.KeyCode.Three,
        [4] = Enum.KeyCode.Four,
    }
    local keypadKeys = {
        [1] = Enum.KeyCode.KeypadOne,
        [2] = Enum.KeyCode.KeypadTwo,
        [3] = Enum.KeyCode.KeypadThree,
        [4] = Enum.KeyCode.KeypadFour,
    }
    snowballHotkey = normalKeys[snowballSlotNumber]
    snowballKeypadHotkey = keypadKeys[snowballSlotNumber]
end

-- Só equipa se o T estiver aberto e a tecla corresponder ao slot real da Snowball.
bind(UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.KeyCode == snowballHotkey or input.KeyCode == snowballKeypadHotkey then
        if isEmotesMenuOpen() then
            equipSnowballAndCloseMenu()
        end
    end
end))

local MAX_SLOTS_PER_ROW = 4

local function getRows(page)
    local rows = {}

    for _, child in ipairs(page:GetChildren()) do
        if child:IsA("GuiObject") then
            local isRow = child.Name:match("^Row%d+$") ~= nil

            if not isRow then
                for _, slot in ipairs(child:GetChildren()) do
                    if slot:IsA("GuiObject") and slot.Name:match("^Emote%d+$") then
                        isRow = true
                        break
                    end
                end
            end

            if isRow then
                table.insert(rows, child)
            end
        end
    end

    table.sort(rows, function(a, b)
        local aNum = tonumber(a.Name:match("^Row(%d+)$"))
        local bNum = tonumber(b.Name:match("^Row(%d+)$"))

        if aNum and bNum and aNum ~= bNum then
            return aNum < bNum
        end

        if a.LayoutOrder ~= b.LayoutOrder then
            return a.LayoutOrder < b.LayoutOrder
        end

        return a.Name < b.Name
    end)

    return rows
end

local function getRowSlots(row)
    local slots = {}

    for _, child in ipairs(row:GetChildren()) do
        if child:IsA("GuiObject") then
            local index = tonumber(child.Name:match("^Emote(%d+)$"))
            if index and index >= 1 and index <= MAX_SLOTS_PER_ROW then
                table.insert(slots, {
                    object = child,
                    index = index,
                    used = child.Visible ~= false,
                })
            end
        end
    end

    table.sort(slots, function(a, b)
        return a.index < b.index
    end)

    return slots
end

local function findFirstFreeSlot(row)
    local used = {}
    local slots = getRowSlots(row)

    for _, info in ipairs(slots) do
        if info.used then
            used[info.index] = true
        end
    end

    for index = 1, MAX_SLOTS_PER_ROW do
        if not used[index] then
            return index, slots
        end
    end

    return nil, slots
end

local function findTemplateSlot(row, wantedIndex, slots)
    -- Se o MM2 já deixou um slot escondido naquela posição, usa ele como molde.
    for _, info in ipairs(slots) do
        if info.index == wantedIndex then
            return info.object
        end
    end

    -- Senão usa o último slot real da mesma linha.
    for i = #slots, 1, -1 do
        if slots[i].object and slots[i].object.Parent then
            return slots[i].object
        end
    end

    return nil
end

local function styleSnowballSlot(snowballSlot, slotNumber)
    snowballSlot.Name = "SnowballSlot"
    snowballSlot:SetAttribute("N4TSnowballSlot", true)
    snowballSlot:SetAttribute("N4TSlotNumber", slotNumber)

    if snowballSlot:IsA("GuiObject") then
        snowballSlot.Visible = true
        snowballSlot.LayoutOrder = slotNumber
    end

    setSnowballHotkey(slotNumber)

    -- Troca apenas o número do atalho pelo número do slot real (1 a 4).
    for _, desc in ipairs(snowballSlot:GetDescendants()) do
        if desc:IsA("TextLabel") then
            local clean = desc.Text:gsub("<.->", ""):gsub("[%s%.]+", "")
            local num = tonumber(clean)
            if num and num >= 0 and num <= 10 then
                desc.RichText = true
                desc.Text = '<font color="#FFFFFF">' .. tostring(slotNumber) .. '</font>'
            end
        end
    end

    local nameLabel = snowballSlot:FindFirstChild("EmoteName")
        or snowballSlot:FindFirstChild("Title")
        or snowballSlot:FindFirstChild("Name")

    if not nameLabel then
        for _, desc in ipairs(snowballSlot:GetDescendants()) do
            if desc:IsA("TextLabel") then
                local clean = desc.Text:gsub("<.->", ""):gsub("[%s%.]+", "")
                if clean ~= "" and tonumber(clean) == nil then
                    nameLabel = desc
                    break
                end
            end
        end
    end

    if nameLabel and nameLabel:IsA("TextLabel") then
        nameLabel.Text = "Snowball"
    end

    local iconLabel = snowballSlot:FindFirstChildWhichIsA("ImageLabel", true)
        or snowballSlot:FindFirstChildWhichIsA("ImageButton", true)

    if iconLabel then
        iconLabel.Image = "rbxthumb://type=Asset&w=150&h=150&id=6123754554"
    end

    local connectedButtons = {}
    local function connectButton(button)
        if connectedButtons[button] then return end
        connectedButtons[button] = true
        bind(button.MouseButton1Click:Connect(equipSnowballAndCloseMenu))
    end

    if snowballSlot:IsA("GuiButton") then
        connectButton(snowballSlot)
    end

    for _, btn in ipairs(snowballSlot:GetDescendants()) do
        if btn:IsA("GuiButton") then
            connectButton(btn)
        end
    end
end

local function removeDuplicateSnowballs(page)
    local found = {}
    for _, obj in ipairs(page:GetDescendants()) do
        if obj.Name == "SnowballSlot" or obj:GetAttribute("N4TSnowballSlot") == true then
            table.insert(found, obj)
        end
    end

    -- Se uma versão antiga deixou várias Snowballs, mantém só uma.
    local keeper = found[1]
    if #found > 1 then
        for i = 2, #found do
            pcall(function() found[i]:Destroy() end)
        end
    end

    -- Também remove linhas extras/vazias que poderiam virar aquelas "25 abas".
    for _, obj in ipairs(page:GetDescendants()) do
        if obj.Name == "SnowballOnlyRow" or obj:GetAttribute("N4TSnowballOnlyRow") == true then
            local keepThisRow = keeper and keeper.Parent and keeper:IsDescendantOf(obj)
            if not keepThisRow then
                pcall(function() obj:Destroy() end)
            end
        end
    end

    return keeper
end

local function injectIntoGameEmotes()
    local page = getGameEmotesPage()
    if not page then
        warn("[Snowball System] Game Emotes page folder not found.")
        return false
    end

    -- Nunca cria outra Snowball se uma já estiver presente.
    local existingSnowball = removeDuplicateSnowballs(page)
    if existingSnowball and existingSnowball.Parent then
        local existingNumber = existingSnowball:GetAttribute("N4TSlotNumber")
        if existingNumber then
            setSnowballHotkey(existingNumber)
        end
        return true
    end

    local rows = getRows(page)
    if #rows == 0 then
        warn("[Snowball System] No existing row found to use as template.")
        return false
    end

    -- Procura primeiro um espaço livre em qualquer linha existente.
    -- Ex.: 3 toys do jogador = Snowball vira o 4º, sem criar outra aba/linha.
    local targetRow
    local freeIndex
    local rowSlots

    for _, row in ipairs(rows) do
        local index, slots = findFirstFreeSlot(row)
        if index then
            targetRow = row
            freeIndex = index
            rowSlots = slots
            break
        end
    end

    if targetRow and freeIndex then
        local template = findTemplateSlot(targetRow, freeIndex, rowSlots)
        if not template then
            warn("[Snowball System] No slot template found in target row.")
            return false
        end

        local snowballSlot = template:Clone()
        styleSnowballSlot(snowballSlot, freeIndex)

        -- Se não existir layout automático, um placeholder escondido na posição
        -- livre já fornece a Position correta porque foi usado como template.
        snowballSlot.Parent = targetRow

        print("[Snowball System] Snowball inserted once in existing row at slot " .. tostring(freeIndex) .. ".")
        return true
    end

    -- Todas as linhas estão cheias. Nesse caso cria UMA única linha adicional,
    -- sem apagar nem limitar os itens reais de quem possui mais toys.
    local sourceRow = rows[#rows]
    local sourceSlots = getRowSlots(sourceRow)
    if #sourceSlots == 0 then
        warn("[Snowball System] Last row has no slot template.")
        return false
    end

    local newRow = sourceRow:Clone()
    newRow.Name = "SnowballOnlyRow"
    newRow:SetAttribute("N4TSnowballOnlyRow", true)
    newRow.Visible = true
    newRow.LayoutOrder = sourceRow.LayoutOrder + 1

    local templateSlot
    for _, child in ipairs(newRow:GetChildren()) do
        if child:IsA("GuiObject") and child.Name:match("^Emote%d+$") then
            if not templateSlot then
                templateSlot = child
            else
                child:Destroy()
            end
        end
    end

    if not templateSlot then
        newRow:Destroy()
        warn("[Snowball System] Could not build Snowball-only row.")
        return false
    end

    styleSnowballSlot(templateSlot, 1)
    newRow.Parent = page

    print("[Snowball System] Existing rows were full; created one Snowball-only row.")
    return true
end

local function ensureSnowballInjected()
    local page = getGameEmotesPage()
    if page then
        local existing = removeDuplicateSnowballs(page)
        if existing and existing.Parent then
            local slotNumber = existing:GetAttribute("N4TSlotNumber")
            if slotNumber then
                setSnowballHotkey(slotNumber)
            end
            return true
        end
    end

    return injectIntoGameEmotes()
end

-- A interface do T pode ainda não existir quando o botão do hub é ligado.
-- Enquanto o recurso estiver ON, tenta até conseguir UMA vez.
task.spawn(function()
    for _ = 1, 20 do
        if ensureSnowballInjected() then
            break
        end
        task.wait(0.5)
    end
end)

-- Se o MM2 reconstruir a interface quando o T abre, recoloca somente se sumiu.
bind(UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.KeyCode == Enum.KeyCode.T then
        task.spawn(function()
            for _, delayTime in ipairs({0.10, 0.25, 0.50, 1.00}) do
                task.wait(delayTime)
                if ensureSnowballInjected() then
                    break
                end
            end
        end)
    end
end))
]==], "@N4T_Snowball")
        if not snowballChunk then
            error(snowballCompileErr or "Falha ao compilar Snowball")
        end
        snowballChunk()
    end)
    if not ok then warn("SNOWBALL:", err) end
    print("✅ SNOWBALL ON")
end
local function snowballOff()
    if _G.N4T_SnowballCleanup then
        pcall(_G.N4T_SnowballCleanup)
        _G.N4T_SnowballCleanup = nil
    end

    -- Fallback de limpeza caso o runtime tenha sido interrompido.
    local backpack = player:FindFirstChild("Backpack")
    if backpack then
        for _, v in ipairs(backpack:GetChildren()) do
            if v.Name == "SnowballToy" or v.Name == "SnowBallToy2020" then
                pcall(function() v:Destroy() end)
            end
        end
    end

    if player.Character then
        for _, v in ipairs(player.Character:GetChildren()) do
            if v.Name == "SnowballToy" or v.Name == "SnowBallToy2020" then
                pcall(function() v:Destroy() end)
            end
        end
    end

    for _, v in ipairs(Workspace:GetChildren()) do
        if v.Name == "SnowballProjectile" or v.Name == "SnowballImpact" then
            pcall(function() v:Destroy() end)
        end
    end

    local pg = player:FindFirstChild("PlayerGui")
    if pg then
        for _, v in ipairs(pg:GetDescendants()) do
            if v.Name == "SnowballOnlyRow" or v.Name == "SnowballSlot" or v:GetAttribute("N4TSnowballOnlyRow") == true or v:GetAttribute("N4TSnowballSlot") == true then
                pcall(function() v:Destroy() end)
            end
        end
    end

    print("❌ SNOWBALL OFF")
end

--========================================================
-- SOMBRAS
--========================================================
local sombrasOriginal
local function sombrasOn()
    sombrasOriginal={Technology=Lighting.Technology,GlobalShadows=Lighting.GlobalShadows,ClockTime=Lighting.ClockTime,GeographicLatitude=Lighting.GeographicLatitude,Brightness=Lighting.Brightness,ExposureCompensation=Lighting.ExposureCompensation,Ambient=Lighting.Ambient,OutdoorAmbient=Lighting.OutdoorAmbient,ShadowSoftness=Lighting.ShadowSoftness,FogStart=Lighting.FogStart,FogEnd=Lighting.FogEnd,FogColor=Lighting.FogColor}
    Lighting.Technology=Enum.Technology.Voxel; Lighting.GlobalShadows=true; Lighting.PrioritizeLightingQuality=true; Lighting.ClockTime=14; Lighting.GeographicLatitude=55; Lighting.Brightness=2; Lighting.ExposureCompensation=0.4; Lighting.Ambient=Color3.fromRGB(120,120,120); Lighting.OutdoorAmbient=Color3.fromRGB(128,128,128); Lighting.ShadowSoftness=0.2; Lighting.FogStart=0; Lighting.FogEnd=100000; Lighting.FogColor=Color3.fromRGB(192,192,192)
    for _,o in ipairs(Lighting:GetChildren()) do if o:IsA("Atmosphere") or o:IsA("BloomEffect") or o:IsA("SunRaysEffect") or o:IsA("ColorCorrectionEffect") or o:IsA("DepthOfFieldEffect") then o:Destroy() end end
    pcall(function() Lighting.EnvironmentDiffuseScale=0; Lighting.EnvironmentSpecularScale=0 end)
    Lighting.Brightness=2; Lighting.ExposureCompensation=0.4; Lighting.GlobalShadows=true
    print("OLD MM2 2024 LIGHTING ativado!")
end
local function sombrasOff()
    if sombrasOriginal then for k,v in pairs(sombrasOriginal) do pcall(function() Lighting[k]=v end) end end
    print("❌ SOMBRAS OFF")
end

--========================================================
-- SOM ABAFADO
--========================================================
local somAbafadoConnection
local function somAbafadoOn()
    local FILTER_NAME = "OldMM2Audio"
    local HIGH_GAIN = -2,1
    local MID_GAIN = -0.7
    local LOW_GAIN = 5
    local function aplicarFiltro(sound)
        if not sound:IsA("Sound") then return end
        if sound:FindFirstChild(FILTER_NAME) then return end
        local equalizer=Instance.new("EqualizerSoundEffect"); equalizer.Name=FILTER_NAME; equalizer.LowGain=LOW_GAIN; equalizer.MidGain=MID_GAIN; equalizer.HighGain=HIGH_GAIN; equalizer.Parent=sound
    end
    for _,object in ipairs(game:GetDescendants()) do aplicarFiltro(object) end
    somAbafadoConnection=game.DescendantAdded:Connect(function(object) task.wait(); aplicarFiltro(object) end)
    print("OLD MM2 AUDIO ativado!")
end
local function somAbafadoOff()
    if somAbafadoConnection then somAbafadoConnection:Disconnect(); somAbafadoConnection=nil end
    for _,object in ipairs(game:GetDescendants()) do if object:IsA("EqualizerSoundEffect") and object.Name=="OldMM2Audio" then object:Destroy() end end
    print("❌ SOM ABAFADO OFF")
end

--========================================================
-- HEADLESS
--========================================================

local headlessCharacterConnection
local headlessDescendantConnection
local headlessSnapshot

local function safeGetTransparency(object)
    local ok, value = pcall(function()
        return object.Transparency
    end)
    if ok then return value end
    return nil
end

local function safeSetTransparency(object, value)
    pcall(function()
        object.Transparency = value
    end)
end

local function restoreHeadless()
    if headlessDescendantConnection then
        pcall(function() headlessDescendantConnection:Disconnect() end)
        headlessDescendantConnection = nil
    end

    local data = headlessSnapshot
    headlessSnapshot = nil
    if not data then return end

    local head = data.head
    if head and head.Parent then
        pcall(function() head.Size = data.size end)
        pcall(function() head.Transparency = data.transparency end)
        pcall(function() head.CanCollide = data.canCollide end)
        pcall(function() head.CanTouch = data.canTouch end)
        pcall(function() head.CanQuery = data.canQuery end)
    end

    for object, transparency in pairs(data.visuals) do
        if object and object.Parent and transparency ~= nil then
            safeSetTransparency(object, transparency)
        end
    end
end

local function applyHeadless(character)
    restoreHeadless()

    local head = character:FindFirstChild("Head") or character:WaitForChild("Head", 10)
    local torso = character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso")
    if not head or not torso then return end

    local visuals = {}
    for _, object in ipairs(head:GetDescendants()) do
        if object:IsA("Decal") or object:IsA("Texture") or object:IsA("SpecialMesh") then
            visuals[object] = safeGetTransparency(object)
        end
    end

    headlessSnapshot = {
        head = head,
        size = head.Size,
        transparency = head.Transparency,
        canCollide = head.CanCollide,
        canTouch = head.CanTouch,
        canQuery = head.CanQuery,
        visuals = visuals
    }

    head.Size = Vector3.new(0.1, 0.1, 0.1)
    head.Transparency = 1
    head.CanCollide = false
    head.CanTouch = false
    head.CanQuery = false

    for object in pairs(visuals) do
        safeSetTransparency(object, 1)
    end

    headlessDescendantConnection = head.DescendantAdded:Connect(function(object)
        if not states["HEADLESS"] then return end
        if object:IsA("Decal") or object:IsA("Texture") or object:IsA("SpecialMesh") then
            if headlessSnapshot and headlessSnapshot.visuals[object] == nil then
                headlessSnapshot.visuals[object] = safeGetTransparency(object)
            end
            safeSetTransparency(object, 1)
        end
    end)
end

local function headlessOff()
    if headlessCharacterConnection then
        pcall(function() headlessCharacterConnection:Disconnect() end)
        headlessCharacterConnection = nil
    end
    restoreHeadless()
    print("❌ HEADLESS OFF")
end

local function headlessOn()
    headlessOff()

    if player.Character then
        task.spawn(function()
            task.wait(0.2)
            if states["HEADLESS"] and player.Character then
                applyHeadless(player.Character)
            end
        end)
    end

    headlessCharacterConnection = player.CharacterAdded:Connect(function(character)
        task.wait(0.2)
        if states["HEADLESS"] then
            applyHeadless(character)
        end
    end)

    print("✅ HEADLESS ON")
end

--========================================================
-- SPEED COIL
--========================================================

local SPEED_COIL_SPEED = 32
local COIL_MESH_ID = "rbxassetid://16606212"
local SPEED_MESH_TEXTURE = "rbxassetid://99170547"
local SPEED_COIL_ICON = "rbxassetid://99170415"
local SPEED_COIL_SOUND = "rbxassetid://99173388"

local speedCoilCharacterConnection
local speedCoilTool
local speedCoilOriginalWalkSpeed
local speedCoilEffectHumanoid

local function removeSpeedCoilEffect()
    if speedCoilEffectHumanoid and speedCoilEffectHumanoid.Parent and speedCoilOriginalWalkSpeed ~= nil then
        pcall(function()
            speedCoilEffectHumanoid.WalkSpeed = speedCoilOriginalWalkSpeed
        end)
    end
    speedCoilEffectHumanoid = nil
    speedCoilOriginalWalkSpeed = nil
end

local function createSpeedCoil()
    local tool = Instance.new("Tool")
    tool.Name = "Zoewar Coil"
    tool.ToolTip = "Speed Up!"
    tool.CanBeDropped = false
    tool.RequiresHandle = true
    tool.TextureId = SPEED_COIL_ICON
    tool:SetAttribute("CoilType", "Speed")

    local handle = Instance.new("Part")
    handle.Name = "Handle"
    handle.Size = Vector3.new(1, 1.2, 2)
    handle.CanCollide = false
    handle.Massless = true
    handle.Parent = tool

    local mesh = Instance.new("SpecialMesh")
    mesh.MeshType = Enum.MeshType.FileMesh
    mesh.MeshId = COIL_MESH_ID
    mesh.TextureId = SPEED_MESH_TEXTURE
    mesh.Scale = Vector3.new(0.7, 0.7, 0.7)
    mesh.Parent = handle

    local sound = Instance.new("Sound")
    sound.Name = "CoilSound"
    sound.SoundId = SPEED_COIL_SOUND
    sound.Volume = 1
    sound.Parent = handle

    tool.Equipped:Connect(function()
        if not states["SPEED COIL"] then return end
        removeSpeedCoilEffect()
        local character = player.Character
        local humanoid = character and character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return end
        speedCoilEffectHumanoid = humanoid
        speedCoilOriginalWalkSpeed = humanoid.WalkSpeed
        humanoid.WalkSpeed = SPEED_COIL_SPEED
        sound:Play()
    end)

    tool.Unequipped:Connect(removeSpeedCoilEffect)
    return tool
end

local function findCoil(coilType)
    local backpack = player:FindFirstChild("Backpack")
    if backpack then
        for _, item in ipairs(backpack:GetChildren()) do
            if item:IsA("Tool") and item:GetAttribute("CoilType") == coilType then
                return item
            end
        end
    end
    if player.Character then
        for _, item in ipairs(player.Character:GetChildren()) do
            if item:IsA("Tool") and item:GetAttribute("CoilType") == coilType then
                return item
            end
        end
    end
    return nil
end

local function destroyCoilsOfType(coilType)
    local backpack = player:FindFirstChild("Backpack")
    if backpack then
        for _, item in ipairs(backpack:GetChildren()) do
            if item:IsA("Tool") and item:GetAttribute("CoilType") == coilType then
                item:Destroy()
            end
        end
    end
    if player.Character then
        for _, item in ipairs(player.Character:GetChildren()) do
            if item:IsA("Tool") and item:GetAttribute("CoilType") == coilType then
                item:Destroy()
            end
        end
    end
end

local function ensureSpeedCoil()
    local found = findCoil("Speed")
    if found then
        speedCoilTool = found
        return
    end
    local backpack = player:FindFirstChild("Backpack") or player:WaitForChild("Backpack", 10)
    if backpack and states["SPEED COIL"] then
        speedCoilTool = createSpeedCoil()
        speedCoilTool.Parent = backpack
    end
end

local function speedCoilOff()
    if speedCoilCharacterConnection then
        pcall(function() speedCoilCharacterConnection:Disconnect() end)
        speedCoilCharacterConnection = nil
    end
    removeSpeedCoilEffect()
    destroyCoilsOfType("Speed")
    speedCoilTool = nil
    print("❌ SPEED COIL OFF")
end

local function speedCoilOn()
    speedCoilOff()
    ensureSpeedCoil()
    speedCoilCharacterConnection = player.CharacterAdded:Connect(function()
        task.wait(0.6)
        if states["SPEED COIL"] then
            ensureSpeedCoil()
        end
    end)
    print("✅ SPEED COIL ON")
end

--========================================================
-- GRAVITY COIL
--========================================================

local GRAVITY_REDUCTION = 0.62
local GRAVITY_MESH_TEXTURE = "rbxassetid://16606141"
local GRAVITY_COIL_ICON = "rbxassetid://16619617"
local GRAVITY_COIL_SOUND = "rbxassetid://16619553"

local gravityCoilCharacterConnection
local gravityCoilTool
local gravityCoilHeartbeat
local gravityCoilForce
local gravityCoilHumanoid
local gravityCoilOldJumpPower
local gravityCoilOldJumpHeight
local gravityCoilUseJumpPower

local function removeGravityCoilEffect()
    if gravityCoilHeartbeat then
        pcall(function() gravityCoilHeartbeat:Disconnect() end)
        gravityCoilHeartbeat = nil
    end
    if gravityCoilForce then
        pcall(function() gravityCoilForce:Destroy() end)
        gravityCoilForce = nil
    end
    if gravityCoilHumanoid and gravityCoilHumanoid.Parent then
        pcall(function()
            if gravityCoilUseJumpPower then
                gravityCoilHumanoid.JumpPower = gravityCoilOldJumpPower
            else
                gravityCoilHumanoid.JumpHeight = gravityCoilOldJumpHeight
            end
        end)
    end
    gravityCoilHumanoid = nil
    gravityCoilOldJumpPower = nil
    gravityCoilOldJumpHeight = nil
    gravityCoilUseJumpPower = nil
end

local function createGravityCoil()
    local tool = Instance.new("Tool")
    tool.Name = "Zoewar Coil"
    tool.ToolTip = "Jump higher!"
    tool.CanBeDropped = false
    tool.RequiresHandle = true
    tool.TextureId = GRAVITY_COIL_ICON
    tool:SetAttribute("CoilType", "Gravity")

    local handle = Instance.new("Part")
    handle.Name = "Handle"
    handle.Size = Vector3.new(1, 1.2, 2)
    handle.CanCollide = false
    handle.Massless = true
    handle.Parent = tool

    local mesh = Instance.new("SpecialMesh")
    mesh.MeshType = Enum.MeshType.FileMesh
    mesh.MeshId = COIL_MESH_ID
    mesh.TextureId = GRAVITY_MESH_TEXTURE
    mesh.Scale = Vector3.new(0.7, 0.7, 0.7)
    mesh.Parent = handle

    local sound = Instance.new("Sound")
    sound.Name = "CoilSound"
    sound.SoundId = GRAVITY_COIL_SOUND
    sound.Volume = 1
    sound.Parent = handle

    tool.Equipped:Connect(function()
        if not states["GRAVITY COIL"] then return end
        removeGravityCoilEffect()

        local character = player.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local root = character:FindFirstChild("HumanoidRootPart")
        if not humanoid or not root then return end

        sound:Play()

        gravityCoilHumanoid = humanoid
        gravityCoilOldJumpPower = humanoid.JumpPower
        gravityCoilOldJumpHeight = humanoid.JumpHeight
        gravityCoilUseJumpPower = humanoid.UseJumpPower

        gravityCoilForce = Instance.new("BodyForce")
        gravityCoilForce.Name = "GravityCoilEffect"
        gravityCoilForce.Force = Vector3.new(0, root.AssemblyMass * workspace.Gravity * GRAVITY_REDUCTION, 0)
        gravityCoilForce.Parent = root

        if gravityCoilUseJumpPower then
            humanoid.JumpPower = 60
        else
            humanoid.JumpHeight = 12
        end

        gravityCoilHeartbeat = RunService.Heartbeat:Connect(function()
            if not states["GRAVITY COIL"]
                or tool.Parent ~= character
                or not character.Parent
                or not humanoid.Parent
                or not root.Parent then
                removeGravityCoilEffect()
                return
            end

            if gravityCoilForce and gravityCoilForce.Parent then
                gravityCoilForce.Force = Vector3.new(0, root.AssemblyMass * workspace.Gravity * GRAVITY_REDUCTION, 0)
            end
        end)
    end)

    tool.Unequipped:Connect(removeGravityCoilEffect)
    return tool
end

local function ensureGravityCoil()
    local found = findCoil("Gravity")
    if found then
        gravityCoilTool = found
        return
    end
    local backpack = player:FindFirstChild("Backpack") or player:WaitForChild("Backpack", 10)
    if backpack and states["GRAVITY COIL"] then
        gravityCoilTool = createGravityCoil()
        gravityCoilTool.Parent = backpack
    end
end

local function gravityCoilOff()
    if gravityCoilCharacterConnection then
        pcall(function() gravityCoilCharacterConnection:Disconnect() end)
        gravityCoilCharacterConnection = nil
    end
    removeGravityCoilEffect()
    destroyCoilsOfType("Gravity")
    gravityCoilTool = nil
    print("❌ GRAVITY COIL OFF")
end

local function gravityCoilOn()
    gravityCoilOff()
    ensureGravityCoil()
    gravityCoilCharacterConnection = player.CharacterAdded:Connect(function()
        task.wait(0.6)
        if states["GRAVITY COIL"] then
            ensureGravityCoil()
        end
    end)
    print("✅ GRAVITY COIL ON")
end

--========================================================
-- VANGUARD HUB
--========================================================
-- O Vanguard precisa rodar como chunk separado porque o código ofuscado
-- usa ... no escopo principal. Mantemos a fonte separada e compilamos no clique.
local vanguardRunning = false
local vanguardSource = [====[
local v0=string.char;local v1=string.byte;local v2=string.sub;local v3=bit32 or bit ;local v4=v3.bxor;local v5=table.concat;local v6=table.insert;local function v7(v24,v25) local v26=0;local v27;while true do if (v26==(695 -(619 + 75))) then return v5(v27);end if (v26==(0 -0)) then v27={};for v108=807 -(118 + 688) , #v24 do v6(v27,v0(v4(v1(v2(v24,v108,v108 + (49 -(25 + 23)) )),v1(v2(v25,1 + 0 + (v108% #v25) ,1 + (v108% #v25) + (1887 -(927 + 959)) )))%256 ));end v26=3 -2 ;end end end local v8=tonumber;local v9=string.byte;local v10=string.char;local v11=string.sub;local v12=string.gsub;local v13=string.rep;local v14=table.concat;local v15=table.insert;local v16=math.ldexp;local v17=getfenv or function() return _ENV;end ;local v18=setmetatable;local v19=pcall;local v20=select;local v21=unpack or table.unpack ;local v22=tonumber;local function v23(v28,v29,...) local v30=(734 -(16 + 716)) -1 ;local v31;v28=v12(v11(v28,5),v7("\204\99","\113\226\77\197\42\188\32"),function(v42) if (v9(v42,3 -1 )==(178 -(11 + 86))) then local v96=0 -0 ;while true do if ((285 -(175 + 110))==v96) then local v126=0 -0 ;while true do if (v126==0) then v31=v8(v11(v42,1,1));return "";end end end end else local v97=0 -0 ;local v98;while true do if (v97==(1796 -(503 + 1293))) then v98=v10(v8(v42,44 -28 ));if v31 then local v132=0 + 0 ;local v133;while true do if (v132==(1062 -(810 + 251))) then return v133;end if (v132==(0 + 0)) then v133=v13(v98,v31);v31=nil;v132=1 + 0 ;end end else return v98;end break;end end end end);local function v32(v43,v44,v45) if v45 then local v99=0 + 0 ;local v100;while true do if (v99==(533 -(43 + 490))) then v100=(v43/((735 -(711 + 22))^(v44-1)))%(2^(((v45-(3 -2)) -(v44-1)) + (860 -(240 + 619)))) ;return v100-(v100%(1 + 0)) ;end end else local v101=0 -0 ;local v102;local v103;while true do if (v101==(1 + 0)) then while true do if (v102==0) then local v135=1744 -(1344 + 400) ;local v136;while true do if (v135==(405 -(255 + 150))) then v136=0 + 0 ;while true do if (v136==(0 + 0)) then v103=(8 -6)^(v44-(3 -2)) ;return (((v43%(v103 + v103))>=v103) and (1740 -(404 + 1335))) or (406 -(183 + 223)) ;end end break;end end end end break;end if (v101==(0 -0)) then v102=0;v103=nil;v101=1;end end end end local function v33() local v46=0;local v47;while true do if (v46==1) then return v47;end if ((0 + 0)==v46) then v47=v9(v28,v30,v30);v30=v30 + 1 + 0 ;v46=338 -(10 + 327) ;end end end local function v34() local v48,v49=v9(v28,v30,v30 + 2 + 0 );v30=v30 + 2 ;return (v49 * (594 -(118 + 220))) + v48 ;end local function v35() local v50=0 + 0 ;local v51;local v52;local v53;local v54;local v55;while true do if (v50==(450 -(108 + 341))) then v53=nil;v54=nil;v50=1 + 1 ;end if (v50==(8 -6)) then v55=nil;while true do local v111=1493 -(711 + 782) ;while true do if (v111==(0 -0)) then if (v51==0) then v52,v53,v54,v55=v9(v28,v30,v30 + 3 );v30=v30 + (473 -(270 + 199)) ;v51=1;end if ((1 + 0)==v51) then return (v55 * (16779035 -(580 + 1239))) + (v54 * (194824 -129288)) + (v53 * 256) + v52 ;end break;end end end break;end if (v50==0) then local v109=0;while true do if (v109==(0 + 0)) then v51=0 + 0 ;v52=nil;v109=1 + 0 ;end if (v109==(2 -1)) then v50=1 + 0 ;break;end end end end end local function v36() local v56=0;local v57;local v58;local v59;local v60;local v61;local v62;while true do local v79=0;while true do if ((1168 -(645 + 522))==v79) then if (v56==(1791 -(1010 + 780))) then local v129=0 + 0 ;local v130;while true do if (v129==0) then v130=0 -0 ;while true do if ((2 -1)==v130) then v56=2;break;end if (v130==(1836 -(1045 + 791))) then v59=2 -1 ;v60=(v32(v58,1,20) * (2^((138 -47) -59))) + v57 ;v130=506 -(351 + 154) ;end end break;end end end if ((1577 -(1281 + 293))==v56) then if (v61==0) then if (v60==(0 -(266 -(28 + 238)))) then return v62 * (0 -0) ;else local v137=1559 -(1381 + 178) ;local v138;while true do if (v137==(0 + 0)) then v138=0 + 0 ;while true do if (v138==(0 + 0)) then v61=3 -1 ;v59=(0 + 0) -(470 -(381 + 89)) ;break;end end break;end end end elseif (v61==(1816 + 231)) then return ((v60==0) and (v62 * ((1 + 0)/0))) or (v62 * NaN) ;end return v16(v62,v61-(1642 -((950 -395) + (1220 -(1074 + 82)))) ) * (v59 + (
