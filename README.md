# Godot Easy Inject

[中文](README.md)|[English](README_en.md)

## 目录

- [介绍](#介绍)
- [为什么选择 Godot Easy Inject?](#为什么选择-godot-easy-inject)
- [安装与启用](#安装与启用)
- [使用方法](#使用方法)
  - [CreateNode 节点自动创建](#createnode-节点自动创建)
  - [NodeService 游戏对象注册](#NodeService-游戏对象注册)
  - [Service 普通类对象](#Service-普通类对象)
  - [依赖注入](#依赖注入)
  - [Node 的命名](#node的命名)
  - [跨场景持久化](#跨场景持久化)
  - [使用容器 API](#使用容器-api)
- [基于里氏替换原则的继承与接口](#基于里氏替换原则的继承与接口)
- [联系方式](#联系方式)

## 介绍

Godot Easy Inject 是一个为 Godot 游戏引擎开发的依赖注入插件，帮助开发者更好地管理游戏组件间的依赖关系，使代码更加模块化、可测试和易于维护。

## 为什么选择 Godot Easy Inject?

在传统 Godot 开发中，获取节点引用通常需要使用 `GetNode<T>(path)` 或导出变量并在编辑器中手动拖拽。例如：

    // 传统方式获取节点引用
    public class Player : Node3D
    {
        // 需要在编辑器中手动拖拽或使用路径查找
        [Export]
        private InventorySystem inventory;

        private GameStateManager gameState;

        public override void _Ready()
        {
            // 硬编码路径获取节点
            gameState = GetNode<GameStateManager>("/root/GameStateManager");
        }
    }

这种方式在大型项目中会导致代码耦合度高、路径变更容易出错，且测试困难。
而使用 Godot Easy Inject，你只需添加几个特性标记，就能实现自动依赖注入：

    [NodeService]
    public class Player : Node3D
    {
        [Inject]
        private InventorySystem inventory;

        [Inject]
        private GameStateManager gameState;

        public override void _Ready()
        {
            // 依赖已注入，直接使用
        }
    }

是否已经等不及想要尝试了呢？现在就开始吧！

## 安装与启用

### 安装插件

从 GitHub 下载插件

在 GitHub 仓库界面点击绿色的 Code 按钮，选择 Download ZIP，下载源码。

解压后将整个 EasyInject 文件夹复制到您的 Godot 项目的 addons 目录中。如果没有 addons 目录，请在项目根目录创建一个。

### 启用插件

在 Godot 编辑器中启用插件

打开 Godot 编辑器，进入项目设置（Project → Project Settings）。

选择 "插件" 选项卡（Plugins），找到 "core_system" 插件，将其状态改为 "启用"（Enable）。

#### 验证插件是否正常工作

插件启用后，只有在运行场景时才会自动初始化 IoC 容器。

要验证插件是否正常工作，您可以创建一个简单的测试脚本并运行场景。

## 使用方法

### CreateNode 节点自动创建

`CreateNode` 特性允许容器自动创建节点实例并注册到IoC容器。

    // 自动创建节点并注册为Node
    [CreateNode]
    public class DebugOverlay : Control
    {
        public override void _Ready()
        {
            // 节点创建逻辑
        }
    }

### NodeService 游戏对象注册

`NodeService `特性用于将场景中已存在的节点注册到IoC容器。

    // 将节点注册为服务
    [NodeService]
    public class Player : CharacterBody3D
    {
        [Inject]
        private GameManager gameManager;

        public override void _Ready()
        {
            // gameManager已注入，可直接使用
        }
    }

### Service 普通类对象

`Service` 特性用于注册普通 C# 类（非 `Node`）服务。

    // 注册普通类为服务
    [Service]
    public class GameManager
    {
        public void StartGame()
        {
            GD.Print("Game started!");
        }
    }

    // 使用自定义名称
    [Service("MainScoreService")]
    public class ScoreService
    {
        public int Score { get; private set; }

        public void AddScore(int points)
        {
            Score += points;
            GD.Print($"Score: {Score}");
        }
    }

### 依赖注入

`Inject` 特性用于标记需要注入的依赖。

    // 服务注入
    [NodeService]
    public class UIController : Control
    {
        // 字段注入
        [Inject]
        private GameManager gameManager;

        // 属性注入
        [Inject]
        public ScoreService ScoreService { get; set; }

        // 带名称的注入
        [Inject("MainScoreService")]
        private ScoreService mainScoreService;

        public override void _Ready()
        {
            gameManager.StartGame();
            mainScoreService.AddScore(100);
        }
    }

    // 构造函数注入 (仅适用于普通类，不适用于Node)
    [Service]
    public class GameLogic
    {
        private readonly GameManager gameManager;
        private readonly ScoreService scoreService;

        // 构造函数注入
        public GameLogic(GameManager gameManager, [Inject("MainScoreService")] ScoreService scoreService)
        {
            this.gameManager = gameManager;
            this.scoreService = scoreService;
        }

        public void ProcessGameLogic()
        {
            gameManager.StartGame();
            scoreService.AddScore(50);
        }
    }

### Node 的命名

Node 可以通过多种方式命名：

    // 默认使用类名
    [NodeService]
    public class Player : Node3D { }

    // 自定义名称
    [NodeService("MainPlayer")]
    public class Player : Node3D { }

    // 使用节点名称
    [NodeService(ENameType.GameObjectName)]
    public class Enemy : Node3D { }

    // 使用字段值
    [NodeService(ENameType.FieldValue)]
    public class ItemSpawner : Node3D
    {
        [NodeName]
        public string SpawnerID = "Level1Spawner";
    }

`NamingStrategy` 枚举提供了以下命名策略选项：

- `Custom`：自定义名称，默认值
- `ClassName`：使用类名作为 Node 名称
- `GameObjectName`：使用节点名称作为 Node 名称
- `FieldValue`：使用标记了 NodeName 的字段值作为 Node 名称

### 跨场景持久化

`PersistAcrossScenes` 特性用于标记在场景切换时不应被销毁的 Node。

    // 持久化的游戏管理器
    [PersistAcrossScenes]
    [Service]
    public class GameProgress
    {
        public int Level { get; set; }
        public int Score { get; set; }
    }

    // 持久化的音频管理器
    [PersistAcrossScenes]
    [NodeService]
    public class AudioManager : Node
    {
        public override void _Ready()
        {
            // 确保不随场景销毁
            GetTree().Root.CallDeferred("add_child", this);
        }

        public void PlaySFX(string sfxPath)
        {
            // 播放音效逻辑
        }
    }

### 使用容器 API

容器提供了以下主要方法，用于手动管理 Node：

    // 获取IoC实例
    var ioc = GetNode("/root/CoreSystem").GetIoC();

    // 获取Node
    var player = ioc.GetNode<Player>();
    var namedPlayer = ioc.GetNode<Player>("MainPlayer");

    // 创建节点Node
    var enemy = ioc.CreateNodeAsNode<Enemy>(enemyResource, "Boss", spawnPoint.Position, Quaternion.Identity);

    // 删除节点Node
    ioc.DeleteNodeNode<Enemy>(enemy, "Boss", true);

    // 清空Node
    ioc.ClearNodesService(); // 清空当前场景的Node
    ioc.ClearNodesService("MainLevel"); // 清空指定场景的Node
    ioc.ClearAllNodesService(true); // 清空所有Node，包括持久化Node

## 基于里氏替换原则的继承与接口

容器支持通过接口或基类实现松耦合依赖注入：

    // 定义接口
    public interface IWeapon
    {
        void Attack();
    }

    // 实现接口的Node
    [NodeService("Sword")]
    public class Sword : Node3D, IWeapon
    {
        public void Attack()
        {
            GD.Print("Sword attack!");
        }
    }

    // 另一个实现
    [NodeService("Bow")]
    public class Bow : Node3D, IWeapon
    {
        public void Attack()
        {
            GD.Print("Bow attack!");
        }
    }

    // 通过接口注入
    [NodeService]
    public class Player : CharacterBody3D
    {
        [Inject("Sword")]
        private IWeapon meleeWeapon;

        [Inject("Bow")]
        private IWeapon rangedWeapon;

        public void AttackWithMelee()
        {
            meleeWeapon.Attack();
        }

        public void AttackWithRanged()
        {
            rangedWeapon.Attack();
        }
    }

当多个类实现同一接口时，需要使用名称区分它们。

## 联系方式

如有任何问题、建议或贡献，请通过 GitHub Issues 提交反馈。
