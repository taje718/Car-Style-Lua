-- Car Style menu (CSP online script for AssettoServer)
--
-- Adds a "Car Style" entry to the in-game online extras menu where a player can:
--   * pick a skin from the skins installed for their car
--   * give parts of the car their own colour (body, hood, doors, bumpers, wheels)
--   * change the window tint
-- Choices are saved per car model, applied locally, and sent to everyone else
-- on the server with an online event, so other players see them too.
--
-- Add to csp_extra_options.ini (next to the overtake script):
--   [SCRIPT_2]
--   SCRIPT = 'https://your-host/carstyle.lua'
--
-- IMPORTANT: which meshes/materials are the hood, doors, wheels etc. is different
-- for every car model. Fill in CAR_CONFIGS below for the cars you care about.
-- Use the "Log mesh names" button in the menu to print the car's meshes and
-- materials to the CSP log (Documents/Assetto Corsa/logs/custom_shaders_patch.log).
-- Cars without an entry use DEFAULT_CONFIG, which is only a guess.

---------------------------------------------------------------------
-- CONFIG
---------------------------------------------------------------------
local CFG = {
  REBROADCAST_S    = 30,    -- re-send your look this often so late joiners get it
  APPLY_THROTTLE_S = 0.25,  -- while dragging a colour, apply at most this often
  SEND_THROTTLE_S  = 0.8,   -- ... and send to other players at most this often
  MAX_TINT         = 0.9,   -- glass darkening at the right end of the slider
}

-- Paintable parts, in the order they are applied. Put broad parts first so that
-- specific parts (hood, doors...) can override them.
local PARTS = {
  { id = 'body',    label = 'Body' },
  { id = 'hood',    label = 'Hood' },
  { id = 'doors',   label = 'Doors' },
  { id = 'fbumper', label = 'Front bumper' },
  { id = 'rbumper', label = 'Rear bumper' },
  { id = 'wheels',  label = 'Wheels / rims' },
}

-- Per car model (folder name in content/cars). Queries are CSP mesh queries,
-- for example 'material:Body' (by material) or a mesh name like 'HOOD'.
--   parts    = which meshes each part colour is applied to
--   glass    = which meshes get the window tint
--   textures = how a skin's texture files map onto the car:
--              { { query = 'material:Body', file = 'body.dds' }, ... }
--              leave out to auto-match every .dds/.png in the skin folder to
--              the material with the same name (works on some cars only)
local CAR_CONFIGS = {
  -- ['my_car_folder'] = {
  --   parts = {
  --     body = 'material:Body', hood = 'HOOD', doors = 'DOOR_*',
  --     fbumper = 'BUMPER_F', rbumper = 'BUMPER_R', wheels = 'material:Rim',
  --   },
  --   glass = 'material:Glass',
  --   textures = {
  --     { query = 'material:Body', file = 'body.dds' },
  --     { query = 'material:Rim',  file = 'rims.dds' },
  --   },
  -- },
}

-- Guess for cars that have no entry above. Verify with the mesh dump.
local DEFAULT_CONFIG = {
  parts = { body = 'material:Body', wheels = 'material:Rim' },
  glass = 'material:Glass',
}

---------------------------------------------------------------------
-- HELPERS
---------------------------------------------------------------------
local function try(fn, ...)
  local ok, res = pcall(fn, ...)
  if not ok then ac.log('carstyle: ' .. tostring(res)) end
  return ok, res
end

local function configFor(carId)
  return CAR_CONFIGS[carId] or DEFAULT_CONFIG
end

local function newLook()
  return { skin = '', tint = 0, colors = {} }   -- colors[partId] = { r, g, b } (0..1)
end

local function isDefaultLook(look)
  if look.skin ~= '' or look.tint > 0.001 then return false end
  for _ in pairs(look.colors) do return false end
  return true
end

-- colours travel over the network as one int each (0 = not customised)
local function packColor(c)
  if not c then return 0 end
  return 0x1000000
    + math.floor(c[1] * 255 + 0.5) * 65536
    + math.floor(c[2] * 255 + 0.5) * 256
    + math.floor(c[3] * 255 + 0.5)
end

local function unpackColor(v)
  if not v or v < 0x1000000 then return nil end
  v = v - 0x1000000
  return { (math.floor(v / 65536) % 256) / 255, (math.floor(v / 256) % 256) / 255, (v % 256) / 255 }
end

local function findMeshes(carIndex, query)
  local ok, res = pcall(function()
    return ac.findNodes('carRoot:' .. carIndex):findMeshes(query)
  end)
  if ok and res then return res end
  return nil
end

---------------------------------------------------------------------
-- SKINS
---------------------------------------------------------------------
local function skinsDir(carId)
  return ac.getFolder(ac.FolderID.ContentCars) .. '/' .. carId .. '/skins'
end

local skinCache = {}
local function listSkins(carId)
  if skinCache[carId] then return skinCache[carId] end
  local out = {}
  local dir = skinsDir(carId)
  local ok, names = pcall(io.scanDir, dir, '*')
  if ok and names then
    for _, n in ipairs(names) do
      if io.dirExists(dir .. '/' .. n) then out[#out + 1] = n end
    end
  end
  table.sort(out)
  skinCache[carId] = out
  return out
end

local function originalSkin(carIndex)
  local ok, id = pcall(ac.getCarSkinID, carIndex)
  if ok and id then return id end
  return ''
end

local autoCache = {}
local function autoTextures(dir)
  if autoCache[dir] then return autoCache[dir] end
  local list = {}
  local ok, files = pcall(io.scanDir, dir, '*')
  if ok and files then
    for _, f in ipairs(files) do
      local base, ext = f:match('^(.+)%.(%w+)$')
      if base and (ext:lower() == 'dds' or ext:lower() == 'png') then
        list[#list + 1] = { query = 'material:' .. base, file = f }
      end
    end
  end
  autoCache[dir] = list
  return list
end

---------------------------------------------------------------------
-- APPLYING A LOOK TO A CAR
---------------------------------------------------------------------
local uniqued = {}   -- [carIndex .. part] = true once materials were made unique

local function applyLook(carIndex, look)
  local carId = ac.getCarID(carIndex)
  if not carId or carId == '' then return end
  local cfg = configFor(carId)

  -- 1. skin textures (this also resets any earlier colours)
  local skin = look.skin ~= '' and look.skin or originalSkin(carIndex)
  if skin ~= '' then
    local dir = skinsDir(carId) .. '/' .. skin
    local list = cfg.textures or autoTextures(dir)
    for _, t in ipairs(list) do
      local meshes = findMeshes(carIndex, t.query)
      if meshes then
        try(function() meshes:setMaterialTexture('txDiffuse', dir .. '/' .. t.file) end)
      end
    end
  end

  -- 2. per-part colours
  if cfg.parts then
    for _, p in ipairs(PARTS) do
      local c = look.colors[p.id]
      local query = cfg.parts[p.id]
      if c and query then
        local meshes = findMeshes(carIndex, query)
        if meshes then
          local key = carIndex .. p.id
          if not uniqued[key] then
            uniqued[key] = true
            -- parts like the hood often share one material with the body
            try(function() meshes:ensureUniqueMaterials() end)
          end
          try(function() meshes:setMaterialTexture('txDiffuse', rgbm(c[1], c[2], c[3], 1)) end)
        end
      end
    end
  end

  -- 3. window tint (darkens the glass)
  if cfg.glass then
    local meshes = findMeshes(carIndex, cfg.glass)
    if meshes then
      local v = 1 - math.min(CFG.MAX_TINT, look.tint * CFG.MAX_TINT)
      try(function() meshes:setMaterialProperty('ksDiffuse', v) end)
    end
  end
end

---------------------------------------------------------------------
-- SAVING
---------------------------------------------------------------------
local store = ac.storage{ data = '' }
local saved = {}   -- [carId] = look

do
  local ok, t = pcall(stringify.parse, store.data)
  if ok and type(t) == 'table' then saved = t end
end

local function saveAll()
  try(function() store.data = stringify(saved, true) end)
end

local function normalise(look)
  look = look or newLook()
  look.skin = look.skin or ''
  look.tint = tonumber(look.tint) or 0
  look.colors = look.colors or {}
  return look
end

local myCarId = ac.getCarID(0) or ''
local myLook = normalise(saved[myCarId])
saved[myCarId] = myLook

---------------------------------------------------------------------
-- SYNC WITH OTHER PLAYERS
---------------------------------------------------------------------
local looks = {}   -- [carIndex] = look received from that player

local layout = {
  key  = ac.StructItem.key('carStyleV1'),
  skin = ac.StructItem.string(48),
  tint = ac.StructItem.float(),
}
for _, p in ipairs(PARTS) do layout['c_' .. p.id] = ac.StructItem.int32() end

local styleEvent = ac.OnlineEvent(layout, function(sender, msg)
  if not sender or sender.index == 0 then return end
  local look = newLook()
  look.skin = msg.skin or ''
  look.tint = msg.tint or 0
  for _, p in ipairs(PARTS) do look.colors[p.id] = unpackColor(msg['c_' .. p.id]) end

  -- nothing to do for a stock look on a car we never changed
  if isDefaultLook(look) and not looks[sender.index] then return end
  looks[sender.index] = look
  try(applyLook, sender.index, look)
end)

local function broadcast()
  if isDefaultLook(myLook) then return end
  local msg = { skin = myLook.skin, tint = myLook.tint }
  for _, p in ipairs(PARTS) do msg['c_' .. p.id] = packColor(myLook.colors[p.id]) end
  try(styleEvent, msg)
end

---------------------------------------------------------------------
-- UPDATE
---------------------------------------------------------------------
local T = {
  startIn = 2.0,         -- apply our saved look shortly after joining
  rebroadcastIn = 4.0,
  applyIn = 0, sendIn = 0,
  dirtyApply = false, dirtySend = false,
}

local function markChanged()
  T.dirtyApply, T.dirtySend = true, true
end

-- when somebody new connects, send them our look
try(function()
  ac.onClientConnected(function()
    T.rebroadcastIn = math.min(T.rebroadcastIn, 5)
  end)
end)

function script.update(dt)
  if T.startIn then
    T.startIn = T.startIn - dt
    if T.startIn <= 0 then
      T.startIn = nil
      if not isDefaultLook(myLook) then try(applyLook, 0, myLook) end
    end
  end

  T.applyIn = T.applyIn - dt
  if T.dirtyApply and T.applyIn <= 0 then
    T.dirtyApply = false
    T.applyIn = CFG.APPLY_THROTTLE_S
    try(applyLook, 0, myLook)
  end

  T.sendIn = T.sendIn - dt
  if T.dirtySend and T.sendIn <= 0 then
    T.dirtySend = false
    T.sendIn = CFG.SEND_THROTTLE_S
    saveAll()
    broadcast()
  end

  T.rebroadcastIn = T.rebroadcastIn - dt
  if T.rebroadcastIn <= 0 then
    T.rebroadcastIn = CFG.REBROADCAST_S
    broadcast()
  end
end

---------------------------------------------------------------------
-- DEBUG: print meshes and materials of the player's car to the CSP log
---------------------------------------------------------------------
local function dumpMeshes()
  local ok, err = pcall(function()
    local all = ac.findNodes('carRoot:0'):findMeshes('?')
    ac.log('carstyle: ---- meshes of ' .. tostring(ac.getCarID(0)) .. ' ----')
    for i = 1, all:size() do
      local m = all:at(i)
      ac.log(string.format('carstyle: mesh=%s  material=%s', tostring(m:name()), tostring(m:getMaterialName())))
    end
    ac.log('carstyle: ---- end ----')
  end)
  if not ok then ac.log('carstyle: mesh dump failed: ' .. tostring(err)) end
end

---------------------------------------------------------------------
-- MENU
---------------------------------------------------------------------
local selectedPart = 1
local pickers = {}   -- [partId] = rgb used by the colour picker

local function setSkin(name)
  myLook.skin = name
  markChanged()
end

local function drawMenu()
  local carId = ac.getCarID(0) or ''
  local cfg = configFor(carId)

  local okName, carName = pcall(ac.getCarName, 0)
  ui.text((okName and carName) or carId)
  ui.separator()

  -- skin
  local skins = listSkins(carId)
  if #skins > 0 then
    local shown = myLook.skin ~= '' and myLook.skin or '(original)'
    ui.combo('Skin', shown, ui.ComboFlags.None, function()
      if ui.selectable('(original)', myLook.skin == '') then setSkin('') end
      for _, s in ipairs(skins) do
        if ui.selectable(s, myLook.skin == s) then setSkin(s) end
      end
    end)
  else
    ui.text('No skins found for this car on your PC.')
  end

  -- parts
  local avail = {}
  for _, p in ipairs(PARTS) do
    if cfg.parts and cfg.parts[p.id] then avail[#avail + 1] = p end
  end

  if #avail > 0 then
    ui.separator()
    if selectedPart > #avail then selectedPart = 1 end
    local part = avail[selectedPart]

    ui.combo('Part', part.label, ui.ComboFlags.None, function()
      for i, p in ipairs(avail) do
        if ui.selectable(p.label, i == selectedPart) then selectedPart = i end
      end
    end)

    local c = myLook.colors[part.id]
    if ui.checkbox('Custom colour##' .. part.id, c ~= nil) then
      if c then
        myLook.colors[part.id] = nil
      else
        myLook.colors[part.id] = { 1, 1, 1 }
        pickers[part.id] = rgb(1, 1, 1)
      end
      markChanged()
      c = myLook.colors[part.id]
    end

    if c then
      local pick = pickers[part.id]
      if not pick then pick = rgb(c[1], c[2], c[3]); pickers[part.id] = pick end
      if ui.colorPicker('##pick' .. part.id, pick, ui.ColorPickerFlags.NoAlpha) then
        myLook.colors[part.id] = { pick.r, pick.g, pick.b }
        markChanged()
      end
    end
  else
    ui.separator()
    ui.text('No paintable parts set up for this car yet.')
  end

  -- window tint
  if cfg.glass then
    ui.separator()
    local v, changed = ui.slider('Window tint', myLook.tint, 0, 1, '%.2f')
    if changed then
      myLook.tint = v
      markChanged()
    end
  end

  ui.separator()
  if ui.button('Reset all') then
    myLook.skin, myLook.tint, myLook.colors = '', 0, {}
    pickers = {}
    markChanged()
  end
  ui.sameLine()
  if ui.button('Log mesh names') then dumpMeshes() end
  if ui.itemHovered() then
    ui.setTooltip('Writes this car\'s mesh and material names to the CSP log, to fill in CAR_CONFIGS.')
  end
end

local menuErrorLogged = false
ui.registerOnlineExtra(
  ui.Icons.Settings or ui.Icons.Menu,
  'Car Style',
  function() return true end,
  function()
    local ok, err = pcall(drawMenu)
    if not ok and not menuErrorLogged then
      menuErrorLogged = true
      ac.log('carstyle: menu error: ' .. tostring(err))
    end
  end,
  nil,
  ui.OnlineExtraFlags.Tool
)
