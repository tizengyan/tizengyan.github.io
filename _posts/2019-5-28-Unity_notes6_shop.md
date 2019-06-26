---
layout: post
title:  "Unity使用笔记6——商店与背包"
#date:   2019-03-01 8:10:54
categories: 游戏开发
tags: unity
excerpt: 商店与背包系统实现
author: Tizeng
---

* content
{:toc}

Unity中的不同场景由不同的Scene组成，而我们在不同的Scene之间切换的时候会丢失前一个Scene的数据，但某些数据比如积分、金币、还剩多少条命需要在游戏中一直记录和保留，这也是要实现商店关卡的关键，因为要读取之前所持有的金币，并在花费过后记录在系统中。可能的方法有三种：1.把需要的数据记录在外部文件中，按需读取；2.使用静态类；3.使用`DontDestroyOnLoad`，标记不希望被销毁的对象。

信息储存的教程来自YouTube博主quill18creates，[地址在此](https://www.youtube.com/watch?v=WchH-JCwVI8)。

## PlayerPrefs

这是Unity中可以用来储存数据的一个接口，我们不用管它是如何储存或是储存在哪里，甚至无关游戏运行在哪个平台，只要调用这个类下面的方法，我们就可以储存或读取数据。注意这个文件中保存的数据甚至不会因为退出程序而销毁。

## 背包

首先在制作背包的界面时一定要搞清楚层级划分，依次为：背包背景、背包（slot holder）、格子（slots）、物品（item），它们依次为包含关系，层层递进。

下面这个脚本实现拖拽，要注意鼠标位置坐标与世界坐标的转换，稍有不慎物品坐标的赋值就会飞出去。

```c#
using UnityEngine;
using UnityEngine.EventSystems;

public class DragHandler : MonoBehaviour, IDragHandler, IEndDragHandler {

    public void OnDrag(PointerEventData eventData) {
        Vector3 mousePos = Input.mousePosition;
        mousePos.z = 10; // 有Vector2转为Vector3，并补全Z轴坐标
        mousePos = Camera.main.ScreenToWorldPoint(mousePos); // 从Input获得的鼠标坐标是基于屏幕的，我们要将其转换成世界坐标
        transform.position = mousePos;
    }

    public void OnEndDrag(PointerEventData eventData) {
        transform.localPosition = Vector3.zero;
    }
}
```

简单的拖拽实现后，就要实现不同格子间的拖放，以及拖放后的信息变更，这在后面一篇博客中具体介绍（UI）。

下面是背包系统的实现，为了让背包中的道具能跨越场景一直存在，我们需要一个不会在场景结束时被销毁的类，且将其实现为一个单例，具体请看`Inventory`脚本中的`Awake`方法。这样我们就可以将道具信息存进一个 List 并维护了。

在我的配置中 GameManager 并不是单例且会被销毁，因此在每个场景初始化时它会一并初始化背包信息。显示道具的思路是，先创建一个prefab，它附有拖动脚本和图标，遍历`Inventory`中储存的所有道具，每有一个道具就生成一个这个prefab，并将道具的图标传入，然后把它放在最近的一个空格上（根据拖拽脚本，需要一并设置好parent），这样玩家就能在背包界面看到道具并用UI的逻辑对其进行交互。

```c#
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class Inventory : MonoBehaviour {

    public GameObject leftSlotsParent;
    public GameObject ItemPrefab;
    public int score = 0;
    public int InventorySize = 16;
    public List<Item> items = new List<Item>();  // inventory info
    public Item bd1;
    public Item bd2;
    public Item bd3;
    public Item bd4;
    public Item bd5;
    public Item bd6;

    public delegate void OnItemChanged();
    public OnItemChanged callback;

    static Inventory instance;
    GameObject[] slots;

    private void Awake() {
        if (instance != null) {
            Destroy(this.gameObject);
            return;
        }
        instance = this;
        GameObject.DontDestroyOnLoad(this.gameObject);
    }

    public static Inventory GetInstance() {
        return instance;
    }

    public void Add(Item item) {
        if(items.Count >= InventorySize) {
            Debug.Log("Inventory Full!");
            return;
        }
        items.Add(item);
        if (callback != null)
            callback.Invoke();
    }

    public void showItems() {
        // 将放入背包的物品显示在UI上
        foreach (Item item in items) {
            GameObject emptySlot = getEmptySlot();
            emptySlot.GetComponent<DropHandler>().isOccupied = true;
            GameObject clone = Instantiate(ItemPrefab, emptySlot.transform.position, emptySlot.transform.rotation);
            clone.transform.SetParent(emptySlot.transform);
            clone.transform.localScale = new Vector2(3, 2);
            clone.GetComponent<Image>().sprite = item.icon;
            clone.GetComponent<DragHandler>().item = item;
        }
    }

    GameObject getEmptySlot() {
        leftSlotsParent = GameObject.FindGameObjectWithTag("LeftSlot");
        GameObject cur = leftSlotsParent.transform.GetChild(0).gameObject;
        if (cur.GetComponent<DropHandler>().isOccupied == true) {
            for (int i = 1; i < 8; i++) {
                cur = leftSlotsParent.transform.GetChild(i).gameObject;
                if (cur.GetComponent<DropHandler>().isOccupied == false)
                    break;
            }
        }

        return cur;
    }

    public void Remove(Item item) {
        items.Remove(item);
    }
}
```

有了储存道具信息的方法，我们还需要定义一个类`Item`来专门存放道具信息，将其定继承`ScriptableObject`，让我们可以在外部新建道具并给属性赋值。

```c#
using UnityEngine;

[CreateAssetMenu(fileName = "new item", menuName = "Inventory/Item")]
public class Item : ScriptableObject {

    public Sprite icon = null;

    public float healthBuff;
    public float damageBuff;
    public float attackSpeedBuff;
    public float bulletSpeedBuff;
    public float strikeRateBuff;
    public float sheildBuff;
    public float cost;
    public string description;

    public Item(float _health, float _damage, float _attackSpeed, float _bulletSpeed, float _strikeRate, float _sheild, float _cost) {
        healthBuff = _health;
        damageBuff = _damage;
        attackSpeedBuff = _attackSpeed;
        bulletSpeedBuff = _bulletSpeed;
        strikeRateBuff = _strikeRate;
        sheildBuff = _sheild;
        cost = _cost;
    }

    public void removeFromInventory() {
        Inventory.GetInstance().Remove(this);
    }
}
```

最后用一个`EquipmentManager`来管理道具和装备道具引起的主角属性变更，它和`Inventory`都被挂在一个不会被销毁的 GameObject 上，在UI界面拖动道具时，会有相应的调用，详见下篇博客。

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class EquipmentManager : MonoBehaviour {

    Inventory inventory;
    const int space = 3;
    static EquipmentManager instance;
    List<Item> currentEquipped;  // equipped info

    private void Awake() {
        if (instance == null)
            instance = this;
    }

    public static EquipmentManager GetInstance() {
        return instance;
    }

    public List<Item> getEquipped() {
        return currentEquipped;
    }

    void Start() {
        inventory = Inventory.GetInstance();
        initialize();
    }

    public void initialize() {
        currentEquipped = new List<Item>();
    }

    public void equip(Item item) {
        if (currentEquipped.Count < space && !currentEquipped.Contains(item)) {
            currentEquipped.Add(item);
            // 装备生效
            FindObjectOfType<PlayerMover>().health += item.healthBuff;
            FindObjectOfType<PlayerMover>().maxHealth += item.healthBuff;
            FindObjectOfType<PlayerMover>().damage += item.damageBuff;
            FindObjectOfType<BulletSpawner>().attackSpeed *= (1 + item.attackSpeedBuff);
            FindObjectOfType<PlayerMover>().attackSpeed *= (1 + item.attackSpeedBuff);
            FindObjectOfType<PlayerMover>().bulletSpeed *= (1 + item.bulletSpeedBuff);
            FindObjectOfType<PlayerMover>().strikeRate += item.strikeRateBuff;
            FindObjectOfType<PlayerMover>().sheildVal += item.sheildBuff;
        }
    }

    public void unequip(Item item) {
        if (currentEquipped.Contains(item)) {
            currentEquipped.Remove(item);
            //inventory.Add(item);
            // 装备失效
            FindObjectOfType<PlayerMover>().health -= item.healthBuff;
            FindObjectOfType<PlayerMover>().maxHealth -= item.healthBuff;
            FindObjectOfType<PlayerMover>().damage -= item.damageBuff;
            FindObjectOfType<BulletSpawner>().attackSpeed /= (1 + item.attackSpeedBuff);
            FindObjectOfType<PlayerMover>().attackSpeed /= (1 + item.attackSpeedBuff);
            FindObjectOfType<PlayerMover>().bulletSpeed /= (1 + item.bulletSpeedBuff);
            FindObjectOfType<PlayerMover>().strikeRate -= item.strikeRateBuff;
            FindObjectOfType<PlayerMover>().sheildVal -= item.sheildBuff;
        }
    }
}
```

## 商店

`ShopManager`被用来在商店关卡实现商店逻辑，在网易MG项目中，商店是一个单独的场景，每次进入商店会有三个种类的道具，可以根据自身持有的金币购买，然后在关卡开始前选择相应道具装备。前面的背包信息是储存在一个可以跨越场景的GameObject中的，这里也要用到。每次出现的道具从已有的道具池（这里是一个List）中随机抽取，demo中一共有六种道具，每次载入商店关卡都会从六个中随机选三个供玩家购买，购买逻辑为：

    从三个商店格中选择一个道具，被选中的格子发光，期间可以随意切换选择的道具

    点击“购买”按钮，购买当前选中的道具，如果所持金币足够，则该位置的道具消失，花费相应金币，道具出现在背包中

我们把三个格子配置成三个按钮，点击时切换素材，实现点击高亮，并在按钮点击时更新`selectedIndex`。每个格子的下一级（子GameObject）都分别有价格、图标、描述的GameObject，初始化时根据随机的物品信息对他们赋值，然后点击“购买”按钮时根据当前的`selectedIndex`购买相应道具。

```c#
using UnityEngine;
using UnityEngine.SceneManagement;
using UnityEngine.UI;

public class ShopManager : MonoBehaviour {

    public GameObject tableView;
    public GameObject itemsView;
    public GameObject shopSlotHolder;
    public GameObject price1;
    public GameObject Icon1;
    public GameObject price2;
    public GameObject Icon2;
    public GameObject price3;
    public GameObject Icon3;

    private Item[] shopSlots;
    private GameObject[] icons = new GameObject[3];
    [Header("Items")]
    public Item bd1;
    public Item bd2;
    public Item bd3;
    public Item bd4;
    public Item bd5;
    public Item bd6;

    Item[] itemPool;
    int poolSize = 6;

    private void Start() {
        shopSlots = new Item[3];
        itemPool = new Item[poolSize];
        itemPool[0] = bd1;
        itemPool[1] = bd2;
        itemPool[2] = bd3;
        itemPool[3] = bd4;
        itemPool[4] = bd5;
        itemPool[5] = bd6;

        initiateShop();
        icons[0] = Icon1;
        icons[1] = Icon2;
        icons[2] = Icon3;
    }

    void initiateShop() {
        int index1 = Mathf.RoundToInt(Random.Range(0, 6));
        int index2 = index1;
        while(index2 == index1) {
            index2 = Mathf.RoundToInt(Random.Range(0, 6));
        }
        int index3 = index1;
        while (index3 == index1 || index3 == index2) {
            index3 = Mathf.RoundToInt(Random.Range(0, 6));
        }
        shopSlots[0] = itemPool[index1];
        shopSlots[1] = itemPool[index2];
        shopSlots[2] = itemPool[index3];

        // 显示图标
        Icon1.GetComponent<Image>().sprite = shopSlots[0].icon;
        Icon2.GetComponent<Image>().sprite = shopSlots[1].icon;
        Icon3.GetComponent<Image>().sprite = shopSlots[2].icon;

        // 显示价格
        price1.GetComponent<Text>().text = shopSlots[0].cost.ToString();
        price2.GetComponent<Text>().text = shopSlots[1].cost.ToString();
        price3.GetComponent<Text>().text = shopSlots[2].cost.ToString();

        // 显示描述
        shopSlotHolder.transform.GetChild(0).GetChild(2).GetComponent<Text>().text = shopSlots[0].description;
        shopSlotHolder.transform.GetChild(1).GetChild(2).GetComponent<Text>().text = shopSlots[1].description;
        shopSlotHolder.transform.GetChild(2).GetChild(2).GetComponent<Text>().text = shopSlots[2].description;

        // 初始化slot下标
        shopSlotHolder.transform.GetChild(0).gameObject.GetComponent<Slot>().index = 0;
        shopSlotHolder.transform.GetChild(1).gameObject.GetComponent<Slot>().index = 1;
        shopSlotHolder.transform.GetChild(2).gameObject.GetComponent<Slot>().index = 2;
    }

    public int selectedIndex = -1;

    public void buy() {
        if (selectedIndex != -1) {
            if (buy(shopSlots[selectedIndex])) {
                Destroy(icons[selectedIndex]);
                Debug.Log("Destroy icon");
            }
        }
    }

    bool buy(Item item) {
        int currentMoney = PlayerPrefs.GetInt("score", 0);
        if (item.cost <= currentMoney) {
            // 扣钱
            currentMoney -= (int)item.cost;
            PlayerPrefs.SetInt("score", currentMoney);
            // 更新背包
            Inventory.GetInstance().Add(item);
            Debug.Log("Item bought");
            return true;
        }
        else {
            print("Not enough money");
            return false;
        }
    }

    public void showItemsView() {
        tableView.SetActive(false);
        itemsView.SetActive(true);
    }

    public void showTableView() {
        tableView.SetActive(true);
        itemsView.SetActive(false);
    }

    public void backToLevel() {
        print("backing to level...");
        SceneManager.LoadScene("LevelScene");
    }

    public void backToMap() {
        print("Backing to Map");
        SceneManager.LoadScene("MapScene");
    }
}
```

此处实现并没有解决出现重复道具的问题，我认为要解决只需要将道具池储存在跨场景的类中，然后在每次购买后将相应道具从道具池移除。

最后在每个格子GameObject上附上`Slot`脚本，用来记录序号。

```c#
public class Slot : MonoBehaviour {
    public int index;
    public ShopManager shopManager;

    public void select() {
        shopManager.selectedIndex = index;
        Debug.Log("Item select");
    }
}
```

## Debug日志

### (1)Canvas在创建Button后过大

将Canvas的render mode改为Screen Space-Camera，然后把Main Camera拖到Render Camera上。

### (2)拖入拖出道具时增益效果重复

出现在道具在装备栏中换位置时，先前的判断条件是只要进入装备栏就添加增益，现在需要检查该道具是否已经存在于装备道具数组中。

### (3)Scene中的场景存在DontDestroy的GameObject，导致之前拖入的场景中的GameObject丢失

不用拖入的方式，用`FindWithTag`来拿到目标GameObject。

### (4)带有道具死亡后重新进入关卡，再装备原有道具无效，取下道具却减少增益

原因是玩家装备道具的信息储存在`InfoHolder`这个不被摧毁的GameObject中，因此携带道具死亡后并不会清空储存装备道具的数组，而装备道具时只有数组中没有当前道具才会启动增益。解决法是每次载入关卡时先重置`EquipmentManager`中的装备道具数组。
