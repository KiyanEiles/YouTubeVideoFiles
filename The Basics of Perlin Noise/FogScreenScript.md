# The Fog Screen Script used in the video

```lua
local partContainer = workspace:WaitForChild("FogScreenObjects")

local fogScreenSize = 100
local resolution = 100
local frequency = 3
local amplitude = 10

local function getHeight(x, z)
	local noiseHeight = math.noise(
		x / resolution * frequency,
		z / resolution * frequency
	)

	noiseHeight = math.clamp(noiseHeight, -0.5, 0.5) + 0.5
	return noiseHeight
end

for x = 0, fogScreenSize do
	for z = 0, fogScreenSize do
		local part = Instance.new("Part")
		part.Parent = partContainer
		part.Anchored = true
		part.Size = Vector3.new(1, 1, 1)
		
		local height = getHeight(x, z)
		
		part.Position = Vector3.new(x, height * amplitude, z)
		part.Color = Color3.new(height, height, height)
	end
	
	game:GetService("RunService").Heartbeat:Wait()
end
```
