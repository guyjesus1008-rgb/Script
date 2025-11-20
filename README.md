-- Script (ServerScriptService/MarkPositionHandler) — com logs adicionados
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local DataStoreService = game:GetService("DataStoreService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

local markEvent = ReplicatedStorage:WaitForChild("MarkPosition")
local returnEvent = ReplicatedStorage:WaitForChild("ReturnToPosition")
local statusEvent = ReplicatedStorage:WaitForChild("PositionStatus")
local feedbackEvent = ReplicatedStorage:WaitForChild("ActionFeedback")

local ds = DataStoreService:GetDataStore("MarkedPositions_v1")

local markedPositions = {} -- player -> Vector3

local function keyForPlayer(player)
    return "markedPos_" .. tostring(player.UserId)
end

-- Logging helpers
local function now()
    return os.date("%Y-%m-%d %H:%M:%S")
end

local function logInfo(player, action, msg)
    local pname = player and player.Name or "System"
    print(string.format("[%s] [INFO] [%s] [%s] %s", now(), action, pname, tostring(msg or "")))
end

local function logWarn(player, action, msg)
    local pname = player and player.Name or "System"
    warn(string.format("[%s] [WARN] [%s] [%s] %s", now(), action, pname, tostring(msg or "")))
end

-- Função utilitária: valida Vector3 básico (não NaN, não infinito, valores razoáveis)
local function isVector3Valid(v)
    if typeof(v) ~= "Vector3" then return false end
    local maxAbs = 1e6
    if v.X ~= v.X or v.Y ~= v.Y or v.Z ~= v.Z then return false end -- NaN
    if math.abs(v.X) > maxAbs or math.abs(v.Y) > maxAbs or math.abs(v.Z) > maxAbs then return false end
    return true
end

-- Salva com retries usando UpdateAsync e backoff exponencial
local function saveWithRetries(key, data, maxAttempts)
    maxAttempts = maxAttempts or 6
    local attempt = 0
    local waitTime = 1
    while attempt < maxAttempts do
        attempt = attempt + 1
        logInfo(nil, "DataStoreSaveAttempt", ("key=%s attempt=%d"):format(key, attempt))
        local ok, err = pcall(function()
            ds:UpdateAsync(key, function(_old)
                return data
            end)
        end)
        if ok then
            logInfo(nil, "DataStoreSaveSuccess", ("key=%s attempt=%d"):format(key, attempt))
            return true
        end
        logWarn(nil, "DataStoreSaveFail", ("key=%s attempt=%d error=%s"):format(key, attempt, tostring(err)))
        wait(waitTime)
        waitTime = math.min(waitTime * 2, 30) -- cap no backoff
    end
    logWarn(nil, "DataStoreSaveFinalFail", ("key=%s attempts=%d"):format(key, maxAttempts))
    return false
end

-- Carrega posição salvo quando o jogador entrar
Players.PlayerAdded:Connect(function(player)
    logInfo(player, "PlayerAdded", "Carregando posição do DataStore")
    local key = keyForPlayer(player)
    local success, result = pcall(function()
        return ds:GetAsync(key)
    end)

    if success and result then
        if type(result) == "table" and result.x and result.y and result.z then
            local pos = Vector3.new(result.x, result.y, result.z)
            if isVector3Valid(pos) then
                markedPositions[player] = pos
                statusEvent:FireClient(player, true)
                feedbackEvent:FireClient(player, "Posição carregada.")
                logInfo(player, "LoadSuccess", ("pos=%s"):format(tostring(pos)))
                return
            else
                logWarn(player, "LoadInvalidData", ("data=%s"):format(tostring(result)))
            end
        else
            logWarn(player, "LoadUnexpectedFormat", ("result=%s"):format(tostring(result)))
        end
    elseif not success then
        logWarn(player, "LoadError", tostring(result))
    end

    statusEvent:FireClient(player, false)
    logInfo(player, "PlayerAdded", "Nenhuma posição salva")
end)

-- Função que checa se a posição tem chão abaixo (raycast)
local function hasGroundAtPosition(pos)
    local origin = pos + Vector3.new(0, 5, 0)
    local direction = Vector3.new(0, -50, 0)
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {}
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    raycastParams.IgnoreWater = true
    local res = Workspace:Raycast(origin, direction, raycastParams)
    if res and res.Position then
        local dist = origin.Y - res.Position.Y
        if dist <= 60 then -- alguma margem
            return true, res.Position.Y
        end
    end
    return false, nil
end

-- Handler para marcar posição
markEvent.OnServerEvent:Connect(function(player, position)
    logInfo(player, "MarkAttempt", ("received position=%s"):format(tostring(position)))
    if not isVector3Valid(position) then
        feedbackEvent:FireClient(player, "Posição inválida enviada.")
        logWarn(player, "MarkInvalidType", tostring(position))
        return
    end

    local character = player.Character
    if not character or not character.Parent then
        feedbackEvent:FireClient(player, "Espere o personagem carregar antes de marcar.")
        logWarn(player, "MarkNoCharacter", "Esperando Character")
        return
    end

    local hrp = character:FindFirstChild("HumanoidRootPart") or character.PrimaryPart
    if not hrp then
        feedbackEvent:FireClient(player, "Personagem sem HumanoidRootPart.")
        logWarn(player, "MarkNoHRP", "HumanoidRootPart ausente")
        return
    end

    local maxDistance = 2000 -- studs, ajuste conforme desejar
    local dist = (position - hrp.Position).Magnitude
    logInfo(player, "MarkDistance", ("distance=%.2f max=%d"):format(dist, maxDistance))
    if dist > maxDistance then
        feedbackEvent:FireClient(player, ("Posição muito distante (%.0f > %d)"):format(dist, maxDistance))
        logWarn(player, "MarkTooFar", ("distance=%.2f"):format(dist))
        return
    end

    local hasGround, groundY = hasGroundAtPosition(position)
    logInfo(player, "MarkGroundCheck", ("hasGround=%s groundY=%s"):format(tostring(hasGround), tostring(groundY)))
    if not hasGround then
        feedbackEvent:FireClient(player, "Posição sem chão detectado. Escolha um lugar válido.")
        logWarn(player, "MarkNoGround", tostring(position))
        return
    end

    markedPositions[player] = position
    local key = keyForPlayer(player)
    local data = { x = position.X, y = position.Y, z = position.Z }

    feedbackEvent:FireClient(player, "Salvando posição...")
    logInfo(player, "MarkSaveStart", ("key=%s data=%s"):format(key, tostring(data)))
    local saved = saveWithRetries(key, data, 6)
    if saved then
        feedbackEvent:FireClient(player, "Posição salva com sucesso.")
        statusEvent:FireClient(player, true)
        logInfo(player, "MarkSaveSuccess", ("pos=%s"):format(tostring(position)))
    else
        feedbackEvent:FireClient(player, "Erro ao salvar posição (tente novamente).")
        logWarn(player, "MarkSaveFail", ("pos=%s"):format(tostring(position)))
    end
end)

-- Teleporta o jogador imediatamente ao clicar voltar (com validação e ajuste de altura)
returnEvent.OnServerEvent:Connect(function(player)
    logInfo(player, "ReturnAttempt", "Tentando teleportar para posição marcada")
    local pos = markedPositions[player]
    if not pos then
        feedbackEvent:FireClient(player, "Nenhuma posição marcada.")
        logWarn(player, "ReturnNoPosition", "Sem posição marcada")
        return
    end

    local character = player.Character
    if not character or not character.Parent then
        feedbackEvent:FireClient(player, "Aguardando respawn para teleportar...")
        logInfo(player, "ReturnWaitRespawn", "Aguardando CharacterAdded")
        character = player.CharacterAdded:Wait()
    end

    local hrp = character:FindFirstChild("HumanoidRootPart") or character.PrimaryPart
    if not hrp then
        feedbackEvent:FireClient(player, "Personagem sem HumanoidRootPart.")
        logWarn(player, "ReturnNoHRP", "HumanoidRootPart ausente")
        return
    end

    if not isVector3Valid(pos) then
        feedbackEvent:FireClient(player, "Posição salva inválida.")
        logWarn(player, "ReturnInvalidSavedPos", tostring(pos))
        return
    end

    local hasGround, groundY = hasGroundAtPosition(pos)
    logInfo(player, "ReturnGroundCheck", ("hasGround=%s groundY=%s"):format(tostring(hasGround), tostring(groundY)))
    local targetCFrame
    if hasGround then
        local safePos = Vector3.new(pos.X, groundY + 3, pos.Z)
        targetCFrame = CFrame.new(safePos)
        logInfo(player, "ReturnTarget", ("safePos=%s"):format(tostring(safePos)))
    else
        targetCFrame = CFrame.new(pos + Vector3.new(0, 3, 0))
        logWarn(player, "ReturnNoGround", ("using raw pos=%s"):format(tostring(pos)))
    end

    -- teleporte imediato
    hrp.CFrame = targetCFrame
    feedbackEvent:FireClient(player, "Teleportado para posição marcada.")
    logInfo(player, "ReturnSuccess", ("teleportedTo=%s"):format(tostring(targetCFrame.p)))
end)

Players.PlayerRemoving:Connect(function(player)
    logInfo(player, "PlayerRemoving", "Salvando posição (se existir) antes de sair")
    local pos = markedPositions[player]
    if not pos then
        logInfo(player, "PlayerRemoving", "Nenhuma posição para salvar")
        return
    end

    local key = keyForPlayer(player)
    local data = { x = pos.X, y = pos.Y, z = pos.Z }
    logInfo(player, "PlayerRemovingSaveStart", ("key=%s data=%s"):format(key, tostring(data)))
    local saved = saveWithRetries(key, data, 6)
    if saved then
        logInfo(player, "PlayerRemovingSaveSuccess", ("pos=%s"):format(tostring(pos)))
    else
        logWarn(player, "PlayerRemovingSaveFail", ("pos=%s"):format(tostring(pos)))
    end

    markedPositions[player] = nil
end)
