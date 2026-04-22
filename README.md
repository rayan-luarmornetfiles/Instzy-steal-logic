
local MiniSwitch = buildSmallSwitch(StealFrame, false, function(v)
    isInstaStealActive = v
end)

ProximityPromptService.PromptTriggered:Connect(function(prompt, triggeredPlayer)
    if triggeredPlayer ~= player or not isInstaStealActive or isStealing then return end
    isStealing = true

    local holdDuration = prompt.HoldDuration
    local startTime = tick()

    local holdConn
    holdConn = RunService.Heartbeat:Connect(function()
        if not prompt:IsDescendantOf(game) then
            if holdConn then holdConn:Disconnect() end
            isStealing = false
            return
        end

        local elapsed = tick() - startTime
        local timeLeft = holdDuration - elapsed

        if timeLeft <= 2.98 then
            if useLaserCape() then
                task.wait(0.2)
                if isPlayerAtFirstFloor() then
                    moveToClosestSpawnThenOwnSpawn()
                elseif isPlayerAtSecondFloor() then
                    teleportToLaserThenDecorationThenForwardThenSpawn()
                end
            end
            if holdConn then holdConn:Disconnect() end
            isStealing = false
        end
    end)
end)

local isServerHopActive = false
local serverHopThread = nil

local function getServerList()
    local placeId = game.PlaceId
    local servers = {}
    local url = "https://games.roblox.com/v1/games/" .. placeId .. "/servers/Public?sortOrder=Asc&limit=100"
    local ok, response = pcall(function() return game:HttpGet(url) end)
    if not ok or not response then
        warn("Server list request failed")
        return servers
    end

    local parsed
    local ok2, err = pcall(function() parsed = HttpService:JSONDecode(response) end)
    if not ok2 or type(parsed) ~= "table" or type(parsed.data) ~= "table" then
        warn("Failed to decode server list or unexpected format")
        return servers
    end

    for _, server in ipairs(parsed.data) do
        if type(server) == "table" and server.playing and server.maxPlayers and server.id then
            if server.playing < server.maxPlayers and server.id ~= game.JobId then
                table.insert(servers, server.id)
            end
        end
    end
    return servers
end

local function attemptServerHop()
    local serverList = getServerList()
    if #serverList > 0 then
        local target = serverList[math.random(1, #serverList)]
        local ok, err = pcall(function()
            TeleportService:TeleportToPlaceInstance(game.PlaceId, target, player)
        end)
        if not ok then
            warn("Teleport failed: ", tostring(err))
        end
    else
        warn("No available servers found.")
    end
end

local function toggleServerHop(active)
    isServerHopActive = active
    if active then
        if serverHopThread == nil then
            serverHopThread = task.spawn(function()
                while isServerHopActive do
                    local ok, err = pcall(attemptServerHop)
                    if not ok then
                        warn("Server hop attempt error:", tostring(err))
                    end
                    task.wait(6.7) -- haha
                end
                serverHopThread = nil
            end)
        end
    else
        isServerHopActive = false
    end
end

local ESP_Switch = CreateSwitch("ESP (players)", false, function(on)
    if on then enableESP() else disableESP() end
end)

local StealPanel_Switch = CreateSwitch("Insta-Steal Panel", false, function(on)
    StealGui.Enabled = on
end)

local ServerHop_Switch = CreateSwitch("Server Hop", false, function(on)
    toggleServerHop(on)
end)
