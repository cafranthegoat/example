local PlayerService = game.Players
local DatastoreService = game:GetService("DataStoreService")
local SkinStore = DatastoreService:GetDataStore("Skins")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local NonInstance = {}
NonInstance.ServerSkins = {}

local SkinDictionary = require(ReplicatedStorage.Dictionaries.SkinsDictionary)
local Weapons = require(ReplicatedStorage.Dictionaries.Weapons)
local TableTemplate = {}

for i, v in pairs(Weapons) do
	TableTemplate[i] = {
		["Equipped"] = "Default",
		["Owned"] = {}
	}
end

function NonInstance.ClearInventory(player)
	NonInstance.ServerSkins[player.Name] = TableTemplate
	print(NonInstance.ServerSkins[player.Name])
end

function NonInstance.load(player)
	local data
	local success, err = pcall(function()
		data = SkinStore:GetAsync(player.UserId)
	end)
	
	NonInstance.ServerSkins[player.Name] = table.clone(TableTemplate)
	
	if success and data then
		for weaponName, weaponData in pairs(data) do
			local EquippedName = weaponData.Equipped
			if EquippedName ~= "Default" then
				
				if type(EquippedName) == "string" then
					local StringSplit = string.split(EquippedName, " ")
					local First, Second = StringSplit[1], StringSplit[2]

					NonInstance.ServerSkins[player.Name][weaponName]["Equipped"] = {}
					NonInstance.ServerSkins[player.Name][weaponName]["Equipped"][EquippedName] = {
						Primary = SkinDictionary[First],
						Secondary = SkinDictionary[Second]
					}
				end
			end
			
			local OwnedSkins = weaponData.Owned
			for _, SkinName in pairs(OwnedSkins) do
				if type(SkinName) == "string" and SkinName ~= "Default" then
					local StringSplit = string.split(SkinName, " ")
					local First, Second = StringSplit[1], StringSplit[2]
					
					NonInstance.ServerSkins[player.Name][weaponName]["Owned"][SkinName] = {
						Primary = SkinDictionary[First],
						Secondary = SkinDictionary[Second]
					}
				end
			end
		end
		
	else
		warn("Could not load stats for " .. player.Name .. ": " .. (err or "No data"))
	end
	
end

function NonInstance.save(player)
	local PlayerSkins = {}

	for weaponName, weaponData in pairs(NonInstance.ServerSkins[player.Name]) do
		local EquippedTable = weaponData.Equipped
		local EquippedTableKey = nil
		
		if type(EquippedTable) == "table" then
			for i, v in pairs(EquippedTable) do
				EquippedTableKey = i
			end
		end
		
		local OwnedTable = weaponData.Owned
		local SearchTable = {}
		for i, v in pairs(OwnedTable) do
			if type(i) == "string" then
				if i == "Owned" then
					continue
				end
				table.insert(SearchTable, i)
			end
		end
		
		PlayerSkins[weaponName] = {
			Equipped = EquippedTableKey or "Default",
			Owned = SearchTable -- separate array for owned skins
		}

		for skinName, skinTable in pairs(weaponData) do
			if skinName ~= "Equipped" and skinName ~= "Default" and skinName ~= "Owned" and typeof(skinTable) == "table" then
				table.insert(PlayerSkins[weaponName].Owned, skinName)
			end
		end
	end
	
	local success, err = pcall(function()
		-- change 2nd argument:
		-- "PlayerSkins" to save data
		-- {} to reset data
		SkinStore:SetAsync(player.UserId, PlayerSkins) 
	end)
	
	if not success then
		warn("Could not save stats for " .. player.Name .. ": " .. err)
	end
end

NonInstance.TestSkin1 = {
	Primary = SkinDictionary.Black,
	Secondary = SkinDictionary.Flame
}

NonInstance.TestSkin2 = {
	Primary = SkinDictionary.Sky,
	Secondary = SkinDictionary.Lemon
}

function NonInstance.GiveSkin(player, skinKey, weapon)
	local StringSplit = string.split(skinKey, " ")
	local First, Second = StringSplit[1], StringSplit[2]
	
	local skinTable = {
		Primary = SkinDictionary[First],
		Secondary = SkinDictionary[Second]
	}
	
	if not skinTable then
		warn("skintable not found")
		return
	end
	
	local primaryAlias, secondaryAlias
	
	for name, data in pairs(SkinDictionary) do
		if data == skinTable.Primary then
			primaryAlias = name
		elseif data == skinTable.Secondary then
			secondaryAlias = name
		end
	end
	
	NonInstance.ServerSkins[player.Name][weapon]["Owned"][skinKey] = skinTable
	
	NonInstance.EquipSkin(player, skinKey, weapon)
end

function NonInstance.EquipSkin(player, skinKey, weapon)
	local RawEquipped = NonInstance.ServerSkins[player.Name][weapon]["Equipped"]
	
	if RawEquipped ~= "Default" then
		local EquippedSkin
		local EquippedSkinName

		for i, v in pairs(NonInstance.ServerSkins[player.Name][weapon]["Equipped"]) do
			if v ~= "Default" then
				EquippedSkin = v
				EquippedSkinName = i
			end
		end

		local StringSplit = string.split(EquippedSkinName, " ")
		local First, Second = StringSplit[1], StringSplit[2]

		local skinTable = {
			Primary = SkinDictionary[First],
			Secondary = SkinDictionary[Second]
		}

		if not skinTable then
			warn("skintable not found")
			return
		end

		NonInstance.ServerSkins[player.Name][weapon]["Owned"][EquippedSkinName] = skinTable
		
	end
	
	NonInstance.ServerSkins[player.Name][weapon]["Equipped"] = {}
	
	if skinKey == "Default" then
		NonInstance.ServerSkins[player.Name][weapon]["Equipped"] = "Default"
		return false
		
	else
		if NonInstance.ServerSkins[player.Name][weapon]["Owned"][skinKey] then
			NonInstance.ServerSkins[player.Name][weapon]["Owned"][skinKey] = nil
			
			local StringSplit = string.split(skinKey, " ")
			local First, Second = StringSplit[1], StringSplit[2]

			local skinTable = {
				Primary = SkinDictionary[First],
				Secondary = SkinDictionary[Second]
			}

			if not skinTable then
				warn("skintable not found")
				return
			end
			
			NonInstance.ServerSkins[player.Name][weapon]["Equipped"][skinKey] = skinTable
			return true
		else
			warn(player.Name .. "'s " .. skinKey .. " " .. weapon .. " skin isn't in their inventory!")
		end
		
	end
end

function NonInstance.RemoveSkin(player, skinTable, weapon)
	
	if skinTable then
		NonInstance.ServerSkins[player.Name][weapon]["Owned"][skinTable] = nil
	else
		warn(player.Name .. "'s " .. skinTable.Name .. " " .. weapon .. " skin isn't in their inventory!")
	end
end

return NonInstance