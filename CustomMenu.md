```
local GameCustomMenuLayer = class("GameCustomMenuLayer",function ( itemDatas,x,y,width,height,clickCallback,clickCallbackObj)
	return display.newLayer()
end )

function GameCustomMenuLayer:ctor(itemDatas,x,y,width,height,clickCallback,clickCallbackObj)
	if not next(itemDatas or {}) then
		return
	end
	--点击事件回调
	self.clickCallback=clickCallback
	self.clickCallbackObj=clickCallbackObj
	self.itemSpace=4
	--可显示的区域高度
	self.areaHeight=0
    self.allItemHeight=0
	self:setContentSize(width or 0 ,height or 0)
	self:setPosition(x or 0, y or 0)
	self.itemViews={}
    self.itemDatas=itemDatas
    self.itemAnims={}
    self.unitItemPosition_y=0
    self:addItemViews()
    self:registerEvent()
end

--添加选项view
function GameCustomMenuLayer:addItemViews( )
	local itemSize=nil
	local layerSize=self:getContentSize()
	local parent_x,parent_y=self:getPosition()
	self.allItemHeight=0
	for k,v in pairs(self.itemDatas or {}) do
			local tempItem=cc.Sprite:create("bt_game_def_0.png")
			local file = device.writablePath.."plaza/res/GameList/bt_game_"..v.id.."_0.png"
        	if cc.FileUtils:getInstance():isFileExist(file) then
            	tempItem:setTexture("GameList/bt_game_"..v.id.."_0.png")
        	end
			itemSize=tempItem:getContentSize()
			self.itemViews[k]=tempItem
	end
	if itemSize then
		self.allItemHeight=(itemSize.height+self.itemSpace)*(#self.itemViews)
		self.unitItemPosition_y=itemSize.height+self.itemSpace
		local areaWidth=itemSize.width
		self.areaHeight=(itemSize.height+self.itemSpace)*3
		self.clipNode=cc.ClippingRectangleNode:create(cc.rect(0,0,areaWidth,self.areaHeight))
			:addTo(self)
		self.clipNode:setPosition((layerSize.width-areaWidth)/2,(layerSize.height-self.areaHeight)/2)
		self.clipArea=cc.rect(parent_x+(layerSize.width-areaWidth)/2,parent_y+(layerSize.height-self.areaHeight)/2,areaWidth,self.areaHeight)
		local x=itemSize.width/2
		local y=self.areaHeight-(itemSize.height+self.itemSpace)/2
		for k,item in pairs(self.itemViews) do
			item:setAnchorPoint(cc.p(0.5,0.5))
			item:setPosition(cc.p(x,y))
			if self.allItemHeight> self.areaHeight then
				local scale=1-math.abs((y-self.areaHeight/2)/(self.areaHeight*2))
				if scale<0.6 then
					scale=0.6
				end
				item:setScale(scale)
			end
			self.clipNode:addChild(item)
			y=y-itemSize.height-self.itemSpace
		end
	end
end

--item移动一段距离
function GameCustomMenuLayer:updateItemPositionByOffset(offset_y)
	if not self.allItemHeight or not self.areaHeight or self.allItemHeight<=self.areaHeight then
		return
	end
	for k,v in pairs(self.itemViews) do
		local pos_x,pos_y=v:getPosition()
		local itemSize=v:getContentSize()
		local final_y,scale=self:convertY_Scale(itemSize,pos_y+offset_y)
		v:setPosition(cc.p(pos_x,final_y))
		v:setScale(scale)
	end
end

--更新item位置
function GameCustomMenuLayer:updatePositions(positions)
	if self.allItemHeight<=self.areaHeight then
		return
	end
	for viewIndex,y in pairs(positions or {}) do
		if self.itemViews and self.itemViews[viewIndex] then
			local pos_x=self.itemViews[viewIndex]:getPosition()
			local itemSize=self.itemViews[viewIndex]:getContentSize()
			local final_y,scale=self:convertY_Scale(itemSize,y)
			self.itemViews[viewIndex]:setPosition(cc.p(pos_x,final_y))
			self.itemViews[viewIndex]:setScale(scale)
		end
	end
end

--根据目前的位置转为可循环的实际位置
function GameCustomMenuLayer:convertY_Scale(itemSize,y)
	local min=self.areaHeight-self.allItemHeight+(itemSize.height+self.itemSpace)*0.5
	local max=self.allItemHeight-(itemSize.height+self.itemSpace)*0.5
	if y < min then
		local offset=math.abs(y-self.areaHeight)%self.allItemHeight
		if y<0 then
			y=self.areaHeight-offset
		else
			y=self.areaHeight+offset
		end
	elseif y > max then
		y=y%self.allItemHeight
	end
	local scale=1-math.abs((y-self.areaHeight/2)/(self.areaHeight*2))
	if scale<0.6 then
		scale=0.6
	end
	return y,scale
end

--计算被点击的item
function GameCustomMenuLayer:calculateClickItemIndex( x,y )
	local pos=self.clipNode:convertToNodeSpace(cc.p(x,y))
	for k,v in pairs(self.itemViews) do
		local v_x,v_y=v:getPosition()
		local itemSize=v:getContentSize()
		local scale=v:getScale()
		if math.abs(v_y-pos.y)<itemSize.height*0.5*scale and math.abs(v_x-pos.x)<itemSize.width*0.5*scale then
			if scale==1 then
				return k
			else
				if v_y>self.areaHeight/2 then
					--向上滚动一格
					self:setItemAnims(2,(itemSize.height+self.itemSpace)/4)
				else
					--向下滚动一格
					self:setItemAnims(2,-(itemSize.height+self.itemSpace)/4)
				end
				return 
			end
		end
	end
end

--stop anim
function GameCustomMenuLayer:onTouchBegan( x,y )
	self.last_time=self.last_time or 0
	self.effectiveEvent=os.clock()-self.last_time>0.28
	if self.effectiveEvent then
		self.last_time=os.clock()
		self.down_position=cc.p(x,y)
		self.last_position=cc.p(x,y)
		if next(self.itemAnims or {}) then
			self.isItemMoving=true
		end
		self.itemAnims={}
	end
	return true
end

--according the moving distance to calculate the positon of item
function GameCustomMenuLayer:onTouchMoved( x,y )
	local offset_y=y-self.last_position.y
	if math.abs(offset_y)>10 and self.effectiveEvent then
		self.last_position=cc.p(x,y)
		self:updateItemPositionByOffset(offset_y)
		self.isItemMoving=true
	end
	return true
end

--calculate end time、end position、acceleration、direction
--according the acceleration to calculate the distance of item need to move
function GameCustomMenuLayer:onTouchEnded( x,y )
	if self.effectiveEvent then
		local offset_time=(os.clock()-self.last_time)*100
		local offset_y=y-self.down_position.y
		local acceleration=offset_y/offset_time
		if math.abs(offset_y)>10 or self.isItemMoving then
			if math.abs(acceleration) > 14 then
				self:setItemAnims(offset_time,offset_y)
			else
				self:setItemAnims(2,acceleration)
			end
			self.isItemMoving=false
			return true
		end
		--判断item点击事件
		if self.clipArea and cc.rectContainsPoint(self.clipArea,cc.p(x,y)) then
			local index=self:calculateClickItemIndex(x,y)
			if index and self.clickCallback and self.clickCallbackObj then
				self.clickCallback(self.clickCallbackObj,index)
			end
		end
	end
end

--ensure the position、highlight status of items
function GameCustomMenuLayer:onEnter( )
	self:scheduleUpdate(function ( )
    		self:onFrameUpdate()
    	end)
end

--stop the anim and the other operation
function GameCustomMenuLayer:onExit( )
    self:unscheduleUpdate()  
	self.itemAnims={}
end

--reset object
function GameCustomMenuLayer:cleanup( )
	self:removeAllChildren()
	self.itemViews={}
    self.itemDatas={}
    self.itemAnims={}
end

function GameCustomMenuLayer:registerEvent( )
	local this=self
	self:setTouchEnabled(true)
    self:registerScriptTouchHandler(function(eventType, x, y)
			if eventType == "began" and self.clipArea and cc.rectContainsPoint(self.clipArea,cc.p(x,y)) then
				return this:onTouchBegan(x, y)
			elseif eventType == "moved" and self.clipArea and cc.rectContainsPoint(self.clipArea,cc.p(x,y)) then
				return this:onTouchMoved(x, y)
			elseif eventType == "ended" then
				 return this:onTouchEnded(x, y)
			end
		return false
    end)
    self:registerScriptHandler(function(eventType)
		if eventType == "enter" then	
			this:onEnter()
		elseif eventType == "exit" then	
			this:onExit()
		elseif eventType == "cleanup" then	
			this:cleanup()
		end
	end)
end
function GameCustomMenuLayer:setItemAnims(offset_time,offset_y)
	if self.allItemHeight<=self.areaHeight then
		return
	end
	self.itemAnims={}
	for k,v in pairs(self.itemViews or {}) do
		local x,y=v:getPosition()
		local anim={}
		anim.viewId=k
		anim.stime=os.clock()*100
		anim.etime=anim.stime+offset_time*10
		anim.fromy=y
		anim.toy=y+offset_y*4
		local offset=anim.toy%self.unitItemPosition_y
		anim.toy=anim.toy-offset+self.unitItemPosition_y*0.5
		-- local times=anim.toy/self.unitItemPosition_y
		-- --为了保证滑动结束之后总有一个item是在中间
		-- anim.toy=math.ceil(times)*self.unitItemPosition_y-self.unitItemPosition_y*0.5
		anim.duration=offset_time*10
		anim.callback=nil
		self.itemAnims[k]=anim
	end
end

--用于模拟滚动动画
function GameCustomMenuLayer:onFrameUpdate()
	local rm={}
	local now=os.clock()*100
	local update=false
	local positions={}
	for k,v in pairs(self.itemAnims or {}) do
	 	if now >= v.stime then
            local endtime = math.min(now, v.etime);
            local y =  self.easeOutQuad(endtime - v.stime, v.fromy, v.toy - v.fromy, v.duration);
   			positions[v.viewId]=y
            if now >= v.etime then
                table.insert(rm, k)
                if v.callback then
                    v.callback(v.obj)
                end
            end
            if not update then 
        		update=true
        	end
        end
    end
    for i = 1, #rm do
        self.itemAnims[rm[i]] = nil
    end
    if update then
        self:updatePositions(positions)
    end
end

-- 计算从b变化到b+c的值。
-- @param t #number current time (t>=0)
-- @param b #number beginning value (b>=0)
-- @param c #number change in value (c>=0)
-- @param d #number duration (d>0)
-- @return #number the calculated result
function GameCustomMenuLayer.easeOutQuad (t, b, c, d)
    t = t / d
    return - c * t *(t - 2) + b
end

return GameCustomMenuLayer
```
