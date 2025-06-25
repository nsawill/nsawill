# Hey, I am nsawill.

I am 15 years old and I have 3 - 4 years of experience with Roblox Studio and LuaU.

Welcome to my **Roblox scripting portfolio** and here are some of the projects I've built or contributed to.

---
## Contact Info
You can find me on discord by the username **"nsawill"** and same for roblox.

## Featured Projects

### Tower of Trials

**Role:** All roles
**Tools:** Rojo,VSCode

**Features:**
- Modular disaster system for creating unique and powerful disasters easily and efficiently
- Modular part system for making towers out of specific parts that have specific properties, very efficient and easy to use
- Player data such as money, available parts, level, exp, exp multiplier etc

**Disaster Handler Code Snippet:**
```lua
local rep = game.ReplicatedStorage
local disasters = require(rep.Modules.Disasters.Disaster)
local bindables = rep.Events.Bindable


function GetDisaster()
	local probabilities = 0
	for _, disaster in pairs(_G.Disasters) do
		probabilities += disaster.Properties.Probability
	end
	local rand = math.random(1,probabilities)
	
	local index = 0
	for _, disaster in pairs(_G.Disasters) do
		if(rand > index and rand < (disaster.Properties.Probability + index)) then
			return disaster
		else
			index += disaster.Properties.Probability
		end
	end
end


function init()
	_G.Disasters = disasters.LoadDisasters()
	_G.FinishedLoadingDisasters = true
	print(_G.Disasters)
end



bindables.GetRandomDisaster.OnInvoke = GetDisaster
init()
```

**Modular Disaster Code Snippet:**
```lua
local disaster = {}

local BaseProperties ={
	Name="",
	Probability=1,
	DomainMultiplier=1.2,
	Domain=""
}

function disaster.New(_properties,_function)
	local newDisaster = setmetatable({},disaster)
	newDisaster.New = nil
	newDisaster.LoadDisasters = nil
	
	newDisaster.Properties = setmetatable({},BaseProperties)
	for name, property in pairs(_properties) do
		newDisaster.Properties[name] = property
	end
	newDisaster.Folder = game.ReplicatedStorage.Disasters:FindFirstChild(_properties.Name)
	newDisaster.Init = _function
	
	return newDisaster
end
```

**Modular Part Code Snippet:**
```lua
local material = {}

-- Default properties
local BaseProperties = {
	Buoyancy=0,
	Strength=0.5,
	Weight=0.5,
	Name="",
	Cost=0,
	LevelCost=1,
	CustomFunction="",
	Material=Enum.Material.Plastic,
	Resistance=0.5,
	Probability = 100
}

-- Make a new material and assign properties
function material.New(properties)
	local newMaterial = setmetatable({},BaseProperties)
	for name, property in pairs(properties) do
		newMaterial[name] = property
	end
	
	return newMaterial
end

-- Load all available materials from folder into a table with the name of the material as the index
function material.LoadMaterials(materialFolder:Folder)
	local materials = {}
	
	for _, nonLoadedMaterial in pairs(materialFolder:GetChildren()) do
		if(nonLoadedMaterial.Name ~= "Template") then
			local properties = {}
			for name, property in pairs(nonLoadedMaterial:GetAttributes()) do
				properties[name] = property
			end
			
			materials[nonLoadedMaterial.Name] = material.New(properties)
		end
	end
	
	local amountOfMaterials = 0
	for _, mat in pairs(materials) do
		amountOfMaterials += 1
	end
	
	for _, mat in pairs(materials) do
		mat.Probability /= amountOfMaterials
	end
	
	return materials
end

-- Get a random material
function material.GetRandomMaterial(materials)
	local totalProbability = 0
	for _, mat in pairs(materials) do
		totalProbability += mat.Probability
	end
	
	local randomProbability = math.random(1,totalProbability)
	local lastProb = 0
	for _, mat in pairs(materials) do
		if randomProbability >= lastProb and randomProbability <= mat.Probability + lastProb then
			return mat
		end
		lastProb += mat.Probability
	end
end

return material
```

**Player Handler Code Snippet:**
```lua
_G.Players = {}
local dss = game:GetService("DataStoreService")
local analyticsService = game:GetService("AnalyticsService")
local ds = dss:GetDataStore("PlayerDataPrototype4")
local players = game.Players
local defaultData = require(script.DefaultData)
local nonSaveableData = require(script.NonSaveable)
local rep = game.ReplicatedStorage
local events = rep.Events
local config = script.Configuration

local ExpLevelNeededMultiplier = config:GetAttribute("ExpNeededLevelMultiplier")
local MoneyRewardedLevelMultiplier = config:GetAttribute("MoneyRewardedLevelMultiplier")

function PlayerAdded(player)
	local data = ds:GetAsync(player.UserId) or defaultData
	for name, item in pairs(nonSaveableData) do
		data[name] = item
	end
	data.MaxHandStorage = 10
	data.Money = math.random(500,5000)
	data.Level = math.random(1,20)
	_G.Players[player.UserId] = data
	print(data)
	_G.CheckIfPlayerHasPurchasedPasses(player)
	RegisterPlayerCollisionGroups(player)
	
	analyticsService:LogOnboardingFunnelStepEvent(
		player,
		1,
		"Player Joined For first time"
	)
	
	task.wait(2)
	events.Bindable.MakeDraggablePart:Fire(player,workspace.Part)
	
	
	task.spawn(function()
		data.Money += 500
		while _G.Materials == nil do
			task.wait()
		end
		
		local playersMaterialList = {}
		for _, material in pairs(_G.Materials) do
			local levelCost = material.LevelCost
			
			if data.Level >= levelCost then
				table.insert(playersMaterialList,material)
			end
		end
		print(playersMaterialList)
		
		local randomMaterials = events.Bindable.GetRandomMaterialList:Invoke(playersMaterialList,20)
		for index, part in pairs(randomMaterials[2]) do
			part.Parent = game.ReplicatedStorage.LoadedParts
			part:SetAttribute("Price",math.ceil(randomMaterials[1][index].Cost * (part.Size).magnitude / 6.5))
		end

		task.spawn(function()
			local response = events.Remote.SendClientMaterialList:InvokeClient(player,randomMaterials)
			for _, part in pairs(randomMaterials[2]) do
				part:Destroy()
			end
		end)

		task.wait(5)
		for _, part in pairs(randomMaterials[2]) do
			part:Destroy()
		end
	end)



end

function RegisterPlayerCollisionGroups(player)
	local char = player.Character or player.CharacterAdded:Wait()
	for _, part in pairs(char:GetDescendants()) do
		if(part:IsA("BasePart")) then
			part.CollisionGroup = "Players"
		end
	end
	
	player.CharacterAdded:Connect(function()
		local char = player.Character
		for _, part in pairs(char:GetDescendants()) do
			if(part:IsA("BasePart")) then
				part.CollisionGroup = "Players"
			end
		end
	end)
end

function PlayerLeft(player)
	local data = _G.Players[player.UserId]
	
	for name, item in pairs(nonSaveableData) do
		data[name] = item
	end
	
	local index = 1
	for name, item in pairs(data) do
		if defaultData[name] == nil then
			table.remove(data,index)
		end
		index+=1
	end
	
	local success,error = pcall(function()
		ds:SetAsync(player.UserId,data)
	end)

	if(success) then
		print("Data saved successfully")
	else
		print("Failed to save data, error: "..error)
	end
end

function GameClose()
	for _, player in pairs(players:GetChildren()) do
		PlayerLeft(player)
	end
end

function _G.PlayerExpChange(player, amount)
	local data = _G.Players[player.UserId]
	local level = data.Level
	local expNeeded = data.ExpNeeded
	local exp = data.Exp
	local multiplier = data.ExpMultiplier
	
	exp += (amount*multiplier)
	_G.Players[player.UserId].Exp = exp
	
	while exp >= expNeeded do
		local difference = exp - expNeeded
		_G.Players[player.UserId].Level += 1
		_G.Players[player.UserId].Exp = difference
		exp = difference
		_G.Players[player.UserId].ExpNeeded *= ExpLevelNeededMultiplier
		expNeeded = _G.Players[player.UserId].ExpNeeded

		local newLevel = _G.Players[player.UserId].Level
		analyticsService:LogOnboardingFunnelStepEvent(
			player,
			newLevel,
			"Player Reached Level "..newLevel
		)
	end
	events.Remote.ServerPlayerDataChange:FireClient(player,_G.Players[player.UserId])
end

function RetrievePlayerData(player)
	events.Remote.ClientRetrievePlayerData:InvokeClient(player,_G.Players[player.UserId])
end

function CompleteTutorial(player)
	local data = _G.Players[player.UserId]
	
	if(data.CompletedTutorial == false) then
		_G.Players[player.UserId].CompletedTutorial = true
		analyticsService:LogCustomEvent(
			player,
			"Finished Tutorial"
		)
	end
end

function _G.PlayerMoneyChange(player,amount,reason)
	local data = _G.Players[player.UserId]
	local money = data.Money
	
	local flowType = "Sink"
	if(amount > 0) then
		flowType = "Source"
		_G.Players[player.UserId].Money = money + (amount*(MoneyRewardedLevelMultiplier*data.Level))
	else
		_G.Players[player.UserId].Money = money + amount
	end
	
	analyticsService:LogEconomyEvent(
		player,
		Enum.AnalyticsEconomyFlowType[flowType],
		"Money",
		math.abs(amount*(MoneyRewardedLevelMultiplier*data.Level)),
		money + (amount*(MoneyRewardedLevelMultiplier*data.Level)),
		Enum.AnalyticsEconomyTransactionType[reason]
	)
end


-- Hook functions to event triggers
game:BindToClose(GameClose)
players.PlayerAdded:Connect(PlayerAdded)
players.PlayerRemoving:Connect(PlayerLeft)
events.Remote.FinishTutorial.OnServerEvent:Connect(CompleteTutorial)
events.Remote.ClientRetrievePlayerData.OnServerInvoke = RetrievePlayerData
```

**Local Data Handler Code Snippet:**
```lua
local rep = game.ReplicatedStorage
local events = rep:WaitForChild("Events")

-- If any data is changed whenever the servers player data is changed, call the bindable event and send the data to any scripts that require updated data
function DataChange(_data)
	for name, statistic in pairs(_data) do
		if(_data[name] ~= _data[name]) then
			script:FindFirstChild(name):Fire(statistic)
		end
	end
	_G.Data = _data
end

-- When new data is retrieved, update the players data
function RetrieveDataCallback(_data)
	_G.Data = _data
	_G.FinishedLoadingData = true
end

-- Ask to get the data from server
function RetrieveData()
	_G.FinishedLoadingData = false
	events:WaitForChild("Remote"):WaitForChild("ClientRetrievePlayerData"):InvokeServer()
end

-- Get player data and make new bindable events
function init()
	RetrieveData()
	for name, statistic in pairs(_G.Data) do
		local event = Instance.new("BindableEvent")
		event.Name = name
		event.Parent = script
	end
end

-- Hook callback functions
events:WaitForChild("Remote"):WaitForChild("ServerPlayerDataChange").OnClientEvent:Connect(DataChange)
events:WaitForChild("Remote"):WaitForChild("ClientRetrievePlayerData").OnClientInvoke = RetrieveDataCallback

-- Game start
init()
```

**Local Parts/Material Handler Code Snippet:**
```lua
local rep = game:GetService("ReplicatedStorage")
local events = rep:WaitForChild("Events")
local remotes = events:WaitForChild("Remote")
local negativeBalanceBindable = events:WaitForChild("Bindable"):WaitForChild("Local"):WaitForChild("NegativeBalance")
local shop = workspace:WaitForChild("Shop")
local pSpawns = shop:WaitForChild("PotentialSpawnPoints")
local localParts = workspace:WaitForChild("LocalParts")
local pp = script:WaitForChild("ProximityPrompt")
_G.PartStorage = {}
_G.NegativeBalance = 0

function GetRandomRotation()
	return vector.create(math.random(1,360),math.random(1,360),math.random(1,360))
end

function HandleMaterialParts(parts,materials)
	for index, part in pairs(localParts:GetChildren()) do
		part:Destroy()
	end
	
	local spawnList = pSpawns:GetChildren()
	for index, part in pairs(materials[2]) do
		local newPart = part:Clone()
		newPart.Parent = localParts
		
		local randomSpawn = math.random(1,#spawnList)
		newPart.Position = spawnList[randomSpawn].Position
		newPart.Rotation = GetRandomRotation()
		table.remove(spawnList,randomSpawn)
		
		local proximityPrompt = pp:Clone()
		proximityPrompt.Parent = newPart
		proximityPrompt.ObjectText = "Cost - $"..part:GetAttribute("Price")
		
		proximityPrompt.Triggered:Connect(function(plr)
			PickUpParts(newPart,materials[1][index])
		end)
	end
end

function PickUpParts(part,material)
	if _G.Data.MaxHandStorage > #_G.PartStorage then
		if not table.find(_G.PartStorage,part) then
			if (_G.Data.Money-_G.NegativeBalance) >= part:GetAttribute("Price") then
				local char = game.Players.LocalPlayer.Character
				local hum = char:WaitForChild("Humanoid")
				
				table.insert(_G.PartStorage,part)
				part:Destroy()
				_G.NegativeBalance += part:GetAttribute("Price")
				negativeBalanceBindable:Fire(_G.NegativeBalance)
				hum.WalkSpeed = 20 - #_G.PartStorage
				print(_G.PartStorage)
			else
				print("Not enough money")
			end
		else
			print("Already have part in storage")
		end
	else
		print("Ran out of storage")
	end
end


function RecieveMaterials(materials)
	print(materials)
	HandleMaterialParts(workspace:WaitForChild("LocalParts"):GetChildren(),materials)
	return true
end

remotes:WaitForChild("SendClientMaterialList").OnClientInvoke = RecieveMaterials
```

---

### Raiders

**Role:** All roles
**Tools:** N/A

**Features:**
- Ability to join players island and raid them

**Raid Handler Code Snippet:**
```lua
local teleportService = game:GetService("TeleportService")
local dataStoreService = game:GetService("DataStoreService")
local dataStore = dataStoreService:GetDataStore("IslandOwnerIds")
local mps = game:GetService("MarketplaceService")
local events = game:GetService("ReplicatedStorage").Events

local function teleportToIsland(plr)
	local accessCode = teleportService:ReserveServer(102625964672948)
	teleportService:TeleportToPrivateServer(102625964672948,accessCode,{plr})
	dataStore:SetAsync(plr.UserId,{["AccessCode"] = accessCode,["BeingRaided"]=dataStore:GetAsync(plr.UserId)["BeingRaided"]})
end

local function raid(plr,plrToRaid)
	if(mps:UserOwnsGamePassAsync(plr.UserId,958688526) and plrToRaid ~= nil and plr.Name ~= plrToRaid) then
		local accessCode = 0
		local success,error = pcall(function()
			if(dataStore:GetAsync(game.Players:GetUserIdFromNameAsync(plrToRaid))["IsOnline"]) then
				accessCode = dataStore:GetAsync(game.Players:GetUserIdFromNameAsync(plrToRaid))["AccessCode"]
			else
				print("Player not online")
			end
		end)
		if(success and accessCode ~= 0) then
			teleportService:TeleportToPrivateServer(102625964672948,accessCode,{plr})
		end
	else
		print(mps:UserOwnsGamePassAsync(plr.UserId,958688526))
		print(plrToRaid)
		print("Dont own gamepass or player doesnt exist or you chose yourself")
	end
end

events.Raid.OnServerEvent:Connect(raid)
events.TeleportToIsland.OnServerEvent:Connect(teleportToIsland)
```
---

### AI Simulation

**Role:** All roles
**Tools:** N/A

**Features:**
- Large amounts of AIs being simulated

**Video:** https://youtu.be/c_7TKQWT57M

**AI Handler Code Snippet:**
```lua

```


---

## Small Commissions

Small commission code is not shown, only relatively large projects.

My small commissions include and are not limited to:
- Bladeball-esque system
- Pick up and Drop system
- Train that follows bezier curve/track
- Leaderboards
- Daily rewards
- Spinning wheel for rewards
- Inventory system
- Player stats system

# More to come soon as I get more projects to work on.
