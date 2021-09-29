---
title: creator3.3web突破char
date: 2021-09-29 14:49:42
tags:
- ts
- js

---

<!-- more -->



## 一、应用

- Label一直是creator开发的一个痛点，官方的`CacheMode`更新后解决了大部分label渲染优化，但是其中的Char模式局限性比较大，所以准备参考一下官方大大的拓展一个适合自己用的，本拓展针对于单场景，label文字内容庞大，一张1024*1024放不下、列表文字很多的情况。



## 二、拓展

在拓展之前，也是参考了乐府在meeting上分享的文章，他的解决思路是重复利用一张`RenderTexture`缓存，对这张`Texture`进行复用等。因为项目原因、以及引擎版本原因，所以在3.3的基础上修改了，修改前有以下几个想法：

- [x] 是否可以增加`char`缓存的`RenderTexture`
- [ ] 增加后是否会导致滥用而导致资源浪费

1. 首先，综合上面两点，需要重新写一个脚本继承自Label，将游戏中进行Label渲染上的分层，如果是`char`模式，选择当前label属于哪种`ECharCacheType`

   ```ts
   import { _decorator, Component, Node, Label, BitmapFont, CacheMode, SpriteFrame, ImageAsset, Texture2D, dynamicAtlasManager, CCString, Enum, log } from "cc";
   const { ccclass, property } = _decorator;
   enum ECharCacheType {
   	normal = 1, //不要改
   	main,
   	dialog,
   	scrollview,
   }
   /**
    * 增加char模式下多种类型的缓存Texture
    * Name = LabelPlus
    * DateTime = Mon Sep 27 2021 10:24:01 GMT+0800 (中国标准时间)
    * Author = 
    * FileBasename = LabelPlus.ts
    * FileBasenameNoExtension = LabelPlus
    * URL = db://assets/LabelPlus.ts
    * ManualUrl = https://docs.cocos.com/creator/3.3/manual/zh/
    *
    */
   @ccclass("LabelPlus")
   export class LabelPlus extends Label {
   	private _charCacheType: ECharCacheType = ECharCacheType.normal;
   	@property({ type: Enum(ECharCacheType), tooltip: "char模式缓存Txture种类" })
   	public get charCacheType(): ECharCacheType {
   		return this._charCacheType;
   	}
   	public set charCacheType(value: ECharCacheType) {
   		this._charCacheType = value;
   		this.updateRenderData();
   	}
   
   	/** 重写父类，char模式传入此label */
   	protected _applyFontTexture() {
   		this.markForUpdateRenderData();
   		const font = this._font;
   		if (font instanceof BitmapFont) {
   			const spriteFrame = font.spriteFrame;
   			if (spriteFrame && spriteFrame.texture) {
   				this._texture = spriteFrame;
   				this.changeMaterialForDefine();
   				if (this._assembler) {
   					this._assembler.updateRenderData(this);
   				}
   			}
   		} else {
   			if (this.cacheMode === CacheMode.CHAR) {
   				this._letterTexture = this._assembler!.getAssemblerData(this);//fix：在char模式下，传入了此label
   				this._texture = this._letterTexture;
   			} else if (!this._ttfSpriteFrame) {
   				this._ttfSpriteFrame = new SpriteFrame();
   				this._assemblerData = this._assembler!.getAssemblerData();
   				const image = new ImageAsset(this._assemblerData!.canvas);
   				const texture = new Texture2D();
   				texture.image = image;
   				this._ttfSpriteFrame.texture = texture;
   			}
   
   			if (this.cacheMode !== CacheMode.CHAR) {
   				// this._frame._refreshTexture(this._texture);
   				this._texture = this._ttfSpriteFrame;
   			}
   			this.changeMaterialForDefine();
   		}
   	}
   }
   ```

2. 修改引擎中相关label的脚本，（修改前要备份一下引擎源码，别问，问就是被坑了）路径大概为/Applications/CocosCreator/Creator/3.3.1/CocosCreator.app/Contents/Resources/resources/3d/engine/cocos/2d/assembler/label/
   顺着`this._assembler!.getAssemblerData(this);`我们找到letter-font.ts，将刚刚传进的label继续向下传递：

   ```ts
   getAssemblerData(target) {//fix:新增target参数
   		if (!_shareAtlas) {
   			_shareAtlas = new LetterAtlas(_atlasWidth, _atlasHeight);
   		}
   
   		return _shareAtlas.getTexture(target.charCacheType);//fix:将charCacheType继续传递
   	},
   ```

   继续向下传递，找到font-utils.ts，对于LetterAtlas的class修改如下：

   ```ts
   //在FontAtlas类中新增几个属性：
   export class FontAtlas {
   	public letterDefinitions;
   	public texture;
   	public _x = space;
   	public _y = space;
   	public _nextY = space;
   
   	constructor(texture) {
   		this.letterDefinitions = {};
   		this.texture = texture;
   		this._x = 0;
   		this._y = 0;
   		this._nextY = 0;
   	}
   }
   //新增char类型
   export enum ECharCacheType {
   	normal = 1, //不要改
   	main,
   	dialog,
   	scrollview,
   }//fix:增加char几张缓存
   export class LetterAtlas {
   public declare fontDefDictionary: Map<number, FontAtlas>;//修改为RenderTxeture数组
   
   constructor(width: number, height: number) {
   		//TODO: 待优化 fix:先把RenderTexture数组初始化好
     	this.fontDefDictionary = new Map();
   		for (let key in ECharCacheType) {
   			const texture = new LetterRenderTexture();
   			texture.initWithSize(width, height);
   			this.fontDefDictionary.set(Number(ECharCacheType[key]), new FontAtlas(texture));
   		}
   		//TODO: 待优化
   
   		this._halfBleed = bleed / 2;
   		this._width = width;
   		this._height = height;
   		director.on(Director.EVENT_BEFORE_SCENE_LAUNCH, this.beforeSceneLoad, this);
   	}
   //insertLetterTexture添加新参数，charCacheType
   public insertLetterTexture(letterTexture: LetterTexture, charCacheType) {
   		const texture = letterTexture.image;
   		const device = director.root!.device;
   		if (!texture || !this.fontDefDictionary.has(charCacheType) || !device) {
   			return null;
   		}
   
   		const width = texture.width;
   		const height = texture.height;
   
   		if (this._x + width + space > this._width) {
   			this._x = space;
   			this._y = this._nextY;
   		}
   
   		if (this._y + height > this._nextY) {
   			this._nextY = this._y + height + space;
   		}
   
   		if (this._nextY > this._height) {
   			warnID(12100);
   			return null;
   		}
   
   		this.fontDefDictionary.get(charCacheType).texture.drawTextureAt(texture, this._x, this._y);
   
   		this._dirty = true;
   
   		const letterDefinition = new FontLetterDefinition();
   		letterDefinition.u = this._x + this._halfBleed;
   		letterDefinition.v = this._y + this._halfBleed;
   		letterDefinition.texture = this.fontDefDictionary.get(charCacheType).texture;
   		letterDefinition.valid = true;
   		letterDefinition.w = letterTexture.width - bleed;
   		letterDefinition.h = letterTexture.height - bleed;
   		letterDefinition.xAdvance = letterDefinition.w;
   		letterDefinition.offsetY = letterTexture.offsetY;
   
   		this._x += width + space;
   		this.fontDefDictionary.get(charCacheType).addLetterDefinitions(letterTexture.hash, letterDefinition);
   		//将对于的RenderTexture的属性更新一下
   		this.fontDefDictionary.get(charCacheType)._x = this._x;
   		this.fontDefDictionary.get(charCacheType)._y = this._y;
   		this.fontDefDictionary.get(charCacheType)._nextY = this._nextY;
   		/*
           const region = new BufferTextureCopy();
           region.texOffset.x = letterDefinition.offsetX;
           region.texOffset.y = letterDefinition.offsetY;
           region.texExtent.width = letterDefinition.w;
           region.texExtent.height = letterDefinition.h;
           */
   
   		return letterDefinition;
   	}
   //重置时候把RenderTexture数组重置
   public reset() {
   		this._x = space;
   		this._y = space;
   		this._nextY = space;
   
   		for (let value of this.fontDefDictionary.values()) {
   			value.clear();
   		}
   	}
   //修改同reset（）
   public destroy() {
   		this.reset();
   		for (let value of this.fontDefDictionary.values()) {
   			value.texture.destroy();
   			value.texture.texture = null;
   		}
   		this.fontDefDictionary.clear();
   	}
   //新增charCacheType参数
   getTexture(charCacheType) {//fix:新增target参数
   		if (!charCacheType) charCacheType = this._charCacheType;
   		if (this.fontDefDictionary.has(charCacheType)) {
   			return this.fontDefDictionary.get(charCacheType).getTexture();
   		}
   		return this.fontDefDictionary.get(this._charCacheType).getTexture();
   	}
   //换场景清空之前的数据
   public clearAllCache() {
   		this.destroy();
   
   		for (let value of this.fontDefDictionary.values()) {
   			const texture = new LetterRenderTexture();
   			texture.initWithSize(this._width, this._height);
   			value.texture.texture = texture;
   		}
   	}
   //新增charCacheType
   public getLetter(key: string, charCacheType) {
   		if (!charCacheType) charCacheType = this._charCacheType;
   		return this.fontDefDictionary.get(charCacheType).letterDefinitions[key];
   	}
   //新增charCacheType
   	public getLetterDefinitionForChar(char: string, labelInfo: ILabelInfo, charCacheType) {
   		if (!this.fontDefDictionary.has(charCacheType)) {
   			const texture = new LetterRenderTexture();
   			texture.initWithSize(this.width, this.height);
   			this.fontDefDictionary.set(charCacheType, new FontAtlas(texture));
   		}
       //将_x,_y,_nextY属性更新为当前的Renderture对应的属性
   		this._x = this.fontDefDictionary.get(charCacheType)._x;
   		this._y = this.fontDefDictionary.get(charCacheType)._y;
   		this._nextY = this.fontDefDictionary.get(charCacheType)._nextY;
   		const hash = char.charCodeAt(0) + labelInfo.hash;
   		let letter = this.fontDefDictionary.get(charCacheType).letterDefinitions[hash];
   		if (!letter) {
   			const temp = new LetterTexture(char, labelInfo);
   			temp.updateRenderData();
   			letter = this.insertLetterTexture(temp, charCacheType);
   			temp.destroy();
   		}
   		// console.log(this.fontDefDictionary)
   		return letter;
   	}
   }
   ```

   然后就是对bmfontUtils.ts中增加一些参数的传递了,只是增加了comp的参数，并未修改其他内容：

   ```
   updateRenderData(comp: Label) {
   		......
   		this._updateContent(comp);
   
   		......
   	},
   _updateContent(comp) {
   		......
   		this._alignText(comp);
   	},
   _multilineTextWrap(nextTokenFunc: Function, comp) {
   		......
   			const tokenLen = nextTokenFunc(_string, index, textLen,comp);
   		......
   			letterDef = shareLabelInfo.fontAtlas!.getLetterDefinitionForChar(character, shareLabelInfo, comp.charCacheType);
   		......
   			this._recordLetterInfo(letterPosition, character, letterIndex, lineIndex, comp);
   		......
   	},
   _getFirstWordLen(text: string, startIndex: number, textLen: number,comp) {
   		......
   		let letterDef = shareLabelInfo.fontAtlas!.getLetterDefinitionForChar(character, shareLabelInfo, comp.charCacheType);
   		......
   			letterDef = shareLabelInfo.fontAtlas!.getLetterDefinitionForChar(character, shareLabelInfo, comp.charCacheType);
   		......
   	},
   _multilineTextWrapByWord(comp) {
   		return this._multilineTextWrap(this._getFirstWordLen, comp);
   	},
   
   	_multilineTextWrapByChar(comp) {
   		return this._multilineTextWrap(this._getFirstCharLen, comp);
   	},
   _recordLetterInfo(letterPosition: Vec2, character: string, letterIndex: number, lineIndex: number, comp?) {
   		......
   		_lettersInfo[letterIndex].valid = shareLabelInfo.fontAtlas!.getLetter(key, comp.charCacheType).valid;
   		......
   	},
   _alignText(comp) {
   		_textDesiredHeight = 0;
   		_linesWidth.length = 0;
   
   		if (!_lineBreakWithoutSpaces) {
   			this._multilineTextWrapByWord(comp);
   		} else {
   			this._multilineTextWrapByChar(comp);
   		}
   
   		this._computeAlignmentOffset();
   
   		// shrink
   		if (_overflow === Overflow.SHRINK) {
   			if (_fontSize > 0 && this._isVerticalClamp()) {
   				this._shrinkLabelToContentSize(this._isVerticalClamp, comp);
   			}
   		}
   
   		if (!this._updateQuads(comp)) {
   			if (_overflow === Overflow.SHRINK) {
   				this._shrinkLabelToContentSize(this._isHorizontalClamp, comp);
   			}
   		}
   	},
   	_scaleFontSizeDown(fontSize: number, comp) {
   		let shouldUpdateContent = true;
   		if (!fontSize) {
   			fontSize = 0.1;
   			shouldUpdateContent = false;
   		}
   		_fontSize = fontSize;
   
   		if (shouldUpdateContent) {
   			this._updateContent(comp);
   		}
   	},
   
   	_shrinkLabelToContentSize(lambda: Function, comp) {
   		const fontSize = _fontSize;
   
   		let left = 0;
   		let right = fontSize | 0;
   		let mid = 0;
   		while (left < right) {
   			mid = (left + right + 1) >> 1;
   
   			const newFontSize = mid;
   			if (newFontSize <= 0) {
   				break;
   			}
   
   			_bmfontScale = newFontSize / _originFontSize;
   
   			if (!_lineBreakWithoutSpaces) {
   				this._multilineTextWrapByWord(comp);
   			} else {
   				this._multilineTextWrapByChar(comp);
   			}
   			this._computeAlignmentOffset();
   
   			if (lambda(comp)) {
   				right = mid - 1;
   			} else {
   				left = mid;
   			}
   		}
   
   		if (left >= 0) {
   			this._scaleFontSizeDown(left, comp);
   		}
   	},
   	_isHorizontalClamp(comp) {
   		.......
   				const letterDef = shareLabelInfo.fontAtlas!.getLetterDefinitionForChar(letterInfo.char, shareLabelInfo, comp.charCacheType);
   				......
   	},
   	_updateQuads(comp) {
   		......
   		shareLabelInfo.fontAtlas!.getTexture(comp.charCacheType);
   		......
   			const letterDef = shareLabelInfo.fontAtlas!.getLetter(letterInfo.hash, comp.charCacheType);
   		......
   	},
   
   ```

   

3. 当然如果不想修改引擎的可以下载Demo：https://github.com/DreamHeavenyang/creator3.3LabelPlus/tree/main/Demo/testLabel3.3.1

