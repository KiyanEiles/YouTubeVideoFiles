# Temporary location for the unoptimised Dialogue Box Script

```lua
local UIS = game:GetService("UserInputService")
local RNS = game:GetService("RunService")
local NPCFolder = workspace:WaitForChild("DialogueNPCs")

local player = game:GetService("Players").LocalPlayer
local char = player.Character or player.ChildAdded:Wait()
local humanoid = char:WaitForChild("Humanoid")
local camera = workspace:WaitForChild("Camera")

local inputPrompt = script.Parent:WaitForChild("NPCInputPrompt")
local inputHolder = inputPrompt:WaitForChild("Holder")
local keyCodeText = inputHolder:WaitForChild("KeyCodeText")

local dialogueHolder = script.Parent:WaitForChild("Holder")
local continueButton = dialogueHolder:WaitForChild("ContinueButton")
local topText = dialogueHolder:WaitForChild("TopText")
local bottomText = dialogueHolder:WaitForChild("BottomText")
local nameText = dialogueHolder:WaitForChild("NameText")

local optionsHolder = dialogueHolder:WaitForChild("OptionsHolder")
local templateOptionButton = optionsHolder:WaitForChild("TemplateButton")

local closestNPC = nil
local savedDialogue = nil
local lineNumber = 0
local currentTalkingSpeed = 0

local NPCTalkingFlag = false
local inConversationFlag = false 
local showingOptionsFlag = false


--// CONFIGURATION VARIABLES \\--
local DETECTION_RANGE = 20
local CAMERA_FOV_IN_CONVERSATION = 40
local MAX_NPCS_WITHIN_RANGE = 5

local INPUT_KEYCODE = Enum.KeyCode.E
local CONTINUE_KEYCODE = Enum.KeyCode.Space
local PAUSEABLE_CHARACTERS = {",", ".", "?", "!", "-", ";", ":"}


--// UI ANIMATION FUNCTIONS \\--
local function LoadDialogueBox()
	dialogueHolder:TweenPosition(UDim2.new(0.5, 0, 0.95, 0), 'Out', 'Quad', 0.3)
	wait(0.3)
end

local function UnloadDialogueBox()
	dialogueHolder:TweenPosition(UDim2.new(0.5, 0, 1.5, 0), 'In', 'Quad', 0.3)
	wait(0.3)
end

local function LoadOptionButton(optionBox)
	optionBox:TweenPosition(UDim2.new(0, 0, optionBox.Position.Y.Scale, 0), 'Out', 'Quad', 0.1)
	wait(0.1)
end

local function UnloadOptionButton(optionBox)
	optionBox:TweenPosition(UDim2.new(-1, 0, optionBox.Position.Y.Scale, 0), 'In', 'Quad', 0.1)
	wait(0.1)
end

local function MoveContinueButton()
	continueButton:TweenPosition(UDim2.new(0.5, 0, 1.1, 0), 'In', 'Quint', 0.075)
	wait(0.075)

	continueButton:TweenPosition(UDim2.new(0.5, 0, 1, 0), 'In', 'Quint', 0.075)
end


--// DIALOGUE FUNCTIONS \\--
local function DisplayText(text, talkingSpeed, textRow, lineIndex)
	for i = 1, #text[lineIndex] do
		textRow.Text = string.sub(text[lineIndex], 0, i)
		
		local currentCharacter = string.sub(text[lineIndex], i, i)
		for _, pauseableCharacter in pairs(PAUSEABLE_CHARACTERS) do
			if currentCharacter == pauseableCharacter then
				wait(0.5)
				break
			end
		end
		
		wait(talkingSpeed)	
	end
end

local function RemoveOptions(selectedOption)
	UnloadOptionButton(selectedOption)
	selectedOption:Destroy()

	for _, optionButton in pairs(optionsHolder:GetChildren()) do
		if optionButton.Name == "TemplateButton" then continue end

		UnloadOptionButton(optionButton)
		optionButton:Destroy()
	end
	
	showingOptionsFlag = false
end

local function DisplayOptions(optionsTable)
	showingOptionsFlag = true
	
	local index = 1
	for option, newDialogue in pairs(optionsTable) do
		local optionButtonClone = templateOptionButton:Clone()
		optionButtonClone.Parent = optionsHolder
		optionButtonClone.Name = "OptionButton"..index
		
		optionButtonClone:WaitForChild("OptionText").Text = option
		
		local heightPos = 1 - ((index - 1) * (optionButtonClone.Size.Y.Scale + 0.05) + 0.025)
		optionButtonClone.Position = UDim2.new(-1, 0, heightPos, 0)
		LoadOptionButton(optionButtonClone)
		
		index += 1
		
		optionButtonClone.MouseButton1Click:Connect(function()
			-- This is whatever you want to do when a specific button is pressed
			savedDialogue = newDialogue
			lineNumber = 1
			
			RemoveOptions(optionButtonClone)
			ContinueConversation()
		end)
	end
end

local function PrintLine(lineToPrint, talkingSpeed)
	for i = 0, 1, 0.1 do
		continueButton.TextTransparency = i
		RNS.RenderStepped:Wait()
	end
	
	DisplayText(lineToPrint, talkingSpeed, topText, 1)
	
	if lineToPrint[2] then
		DisplayText(lineToPrint, talkingSpeed, bottomText, 2)
	end
	if lineToPrint.Options then
		DisplayOptions(lineToPrint.Options)
	end
		
	for i = 0, 1, 0.1 do
		continueButton.TextTransparency = 1 - i
		RNS.RenderStepped:Wait()
	end
end

local function ClearText()
	topText.Text = ""
	bottomText.Text = ""
end


--// CONVERSATION FUNCTIONS \\--
local function FindClosestNPC()
	local distanceFromNPCs = {}

	for _, _NPC in pairs(NPCFolder:GetChildren()) do
		if #NPCFolder:GetChildren() <= MAX_NPCS_WITHIN_RANGE then
			local _dist = (char.PrimaryPart.Position - _NPC.PrimaryPart.Position).Magnitude
			if _dist <= DETECTION_RANGE then
				local NPCToPlayer = (char.PrimaryPart.Position - _NPC.PrimaryPart.Position).Unit
				local NPCLookVector = _NPC.PrimaryPart.CFrame.LookVector

				local dotProduct = NPCToPlayer:Dot(NPCLookVector)

				if dotProduct > 0.7 then
					table.insert(distanceFromNPCs, {
						NPC = _NPC,
						Distance = _dist
					})
				end
			end
		else
			break
		end
	end

	if #distanceFromNPCs ~= 0 then
		table.sort(distanceFromNPCs, function(a, b)
			return a.Distance < b.Distance
		end)
		closestNPC = distanceFromNPCs[1].NPC
	else
		closestNPC = nil
	end
end

local function RotateTowardsObject(NPCToRotate, targetObject)
	local modifiedPos = Vector3.new(targetObject.Position.X, NPCToRotate.Position.Y, targetObject.Position.Z)
	
	for i = 0, 0.25, 0.01 do
		NPCToRotate.CFrame = NPCToRotate.CFrame:Lerp(CFrame.new(NPCToRotate.Position, modifiedPos), i)
		RNS.Heartbeat:Wait()
	end
end

local function SetupConversation(NPC)
	if not NPCTalkingFlag and not inConversationFlag then
		NPCTalkingFlag = true
		inConversationFlag = true
		
		local NPCSettings = require(NPC:WaitForChild("NPCSettings"))
		savedDialogue = NPCSettings.Dialogue
		currentTalkingSpeed = NPCSettings.TalkingSpeed
		nameText.Text = NPCSettings.Name
		
		humanoid.JumpHeight = 0
		humanoid.WalkSpeed = 0
		
		camera.FieldOfView = CAMERA_FOV_IN_CONVERSATION
		
		RotateTowardsObject(char.PrimaryPart, NPC.PrimaryPart) -- Rotates the NPC and player towards eachother
		RotateTowardsObject(NPC.PrimaryPart, char.PrimaryPart)
		
		--^^ The prep for conversation ^^
		
		local introAnim = Instance.new("Animation")
		introAnim.AnimationId = NPCSettings.IntroAnimationID
		
		local introTrack = NPC:WaitForChild("Humanoid"):LoadAnimation(introAnim)
		introTrack.Looped = false
		introTrack:Play()
		
		ClearText()
		LoadDialogueBox()
		
		lineNumber = 1
		local currentLine = savedDialogue[lineNumber]
		PrintLine(currentLine, currentTalkingSpeed)
		
		lineNumber += 1
		NPCTalkingFlag = false
	end
end

function ContinueConversation()
	if not NPCTalkingFlag and not showingOptionsFlag and inConversationFlag then
		if lineNumber > #savedDialogue then
			--// Resets current NPC data
			savedDialogue = nil
			lineNumber = 0
			currentTalkingSpeed = 0
			
			camera.FieldOfView = 70
			
			NPCTalkingFlag = false
			inConversationFlag = false
			
			UnloadDialogueBox()
			ClearText()
			
			humanoid.JumpHeight = 7.2
			humanoid.WalkSpeed = 16
		else
			--// Prints new line from the dialogue settings
			NPCTalkingFlag = true
			ClearText()
			
			local currentLine = savedDialogue[lineNumber]
			PrintLine(currentLine, currentTalkingSpeed)
			
			lineNumber += 1
			NPCTalkingFlag = false
		end
	end
end


--// EVENTS \\--
UIS.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.Keyboard then
		if input.KeyCode == INPUT_KEYCODE and closestNPC then
			SetupConversation(closestNPC)
		elseif input.KeyCode == CONTINUE_KEYCODE and closestNPC then
			MoveContinueButton()
			ContinueConversation()
		end
	end
end)

RNS.Heartbeat:Connect(function()
	FindClosestNPC()
	
	if closestNPC and not inConversationFlag then
		inputPrompt.Adornee = closestNPC:FindFirstChild("Head")
		inputPrompt.Enabled = true
	else
		inputPrompt.Adornee = nil
		inputPrompt.Enabled = false
	end
end)
```
