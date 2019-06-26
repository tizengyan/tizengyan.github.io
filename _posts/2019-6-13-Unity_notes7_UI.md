---
layout: post
title:  "Unity使用笔记7——UI界面"
#date:   2019-03-01 8:10:54
categories: 游戏开发
tags: unity
excerpt: 部分UI界面实现
author: Tizeng
---

* content
{:toc}

## 拖拽功能

拖拽拖拽，其实是两个功能，一个是拖，一个是放，这里不说拖放是怕误解为将窗口拖大那种拖放。

### 拖（drag）

下面的脚本附在需要被拖动的物体上，注意继承的几个类。主要有两个功能：一是在拖动时将被拖物体与鼠标位置绑定，在2D世界中需要一个屏幕坐标到世界坐标的简单转换；二是在每次拖动完毕后给物件重新设置上级（父）GameObject，由于拖和放是两个相对的动作，unity中的`OnDrop`方法会比`OnEndDrag`先执行，因此我们只要在`OnDrop`中更新上级GameObject，就可以在每次拖动到不同格子时确保上级GameObject也变更了，这样一来只需要将被拖动物体的相对坐标（也就是相对于上级物体的坐标）置为0，它就会显示在其上级物体的中心。

要注意的是在刚开始拖动和结束拖动时注意控制`CanvasGroup`中的`blocksRaycasts`变量，否则unity将检测不出鼠标发出的 raycast，影响功能实现。

```c#
using UnityEngine;
using UnityEngine.EventSystems;

public class DragHandler : MonoBehaviour, IDragHandler, IEndDragHandler, IBeginDragHandler {

    public Transform originalParent = null;
    public Item item;

    public void OnBeginDrag(PointerEventData eventData) {
        originalParent = transform.parent;
        GetComponent<CanvasGroup>().blocksRaycasts = false;
        eventData.pointerDrag.GetComponentInParent<DropHandler>().isOccupied = false;
    }

    public void OnDrag(PointerEventData eventData) {
        Vector3 mousePos = Input.mousePosition;
        mousePos.z = 10;
        mousePos = Camera.main.ScreenToWorldPoint(mousePos);
        transform.position = mousePos;
    }

    public void OnEndDrag(PointerEventData eventData) {
        //transform.localPosition = Vector3.zero;
        transform.SetParent(originalParent);
        eventData.pointerDrag.GetComponentInParent<DropHandler>().isOccupied = true;
        transform.localPosition = Vector3.zero;
        GetComponent<CanvasGroup>().blocksRaycasts = true;
    }
}
```

每个格子只能存在一个道具，因此附在格子上的`DropHandler`中就要有一个变量来记录是否当前格子被占用。

### 放（drop）

类似，下面的脚本被附在需要接受被拖拽的物体上。要注意发生属性变更的情况，只有当从非装备栏到装备栏和从装备栏到非装备栏时，才发生属性变更。

```c#
using UnityEngine;
using UnityEngine.EventSystems;

public class DropHandler : MonoBehaviour, IDropHandler, IPointerEnterHandler {

    public bool isOccupied = false;
    public bool isEquipmentSlot = false;

    public void OnDrop(PointerEventData eventData) {
        if (!isOccupied) {
            Debug.Log("Moving items");
            isOccupied = true;
            eventData.pointerDrag.GetComponent<DragHandler>().originalParent = this.transform;
            bool dragFromEquipment = eventData.pointerDrag.transform.GetComponentInParent<DropHandler>().isEquipmentSlot; // 注意逻辑
            if (isEquipmentSlot && !dragFromEquipment) // 从非装备区到装备区
                EquipmentManager.GetInstance().equip(eventData.pointerDrag.GetComponent<DragHandler>().item);
            else if(!isEquipmentSlot && dragFromEquipment) { // 反之
                EquipmentManager.GetInstance().unequip(eventData.pointerDrag.GetComponent<DragHandler>().item);
            }
        }
    }
}
```

## SceneLoader

有时场景切换需要一些过渡，比如我们想实现一个点击开始后，不是马上载入关卡，而是播放一段载入动画（比如简单的渐隐转场），当动画播放完毕后，再载入关卡，下面的脚本就可以实现这个功能。

首先在Animator中设置一个空的动画，创建一个transition到我们需要播放的目标动画，以及一个 Trigger 类型的参数，然后用一个函数来控制这个 Trigger，这样我们只用在需要切换场景的地方（比如某个 Button 被按下时）调用这个函数，就会激活目标动画的播放。

最后，在目标动画播放完毕的末尾，添加一个 Animation Event，将载入目标场景的函数设为将被激活的 Event，这样在动画播放完毕后就会自动调用载入场景的函数了。

下面是实例脚本：

```c#
using UnityEngine;
using UnityEngine.SceneManagement;

public class SceneLoader : MonoBehaviour {
    Animator animator;
    public string sceneName;
    static SceneLoader instance;

    public static SceneLoader GetInstance() {
        return instance;
    }

    private void Start() {
        if (instance == null)
            instance = this;
        animator = GetComponent<Animator>();
    }

    public void fadeToLevel() {
        animator.SetTrigger("Trigger");
    }

    public void OnFadeComplete() {
        SceneManager.LoadScene(sceneName);
    }

    public void toStart() {
        SceneManager.LoadScene("StartScene");
    }
}
```

## 滚动菜单

下面这个脚本实现可以横向拖动的菜单或任何游戏中的物件，首先定义一个判断有无点击的变量，在`Update`方法中，如果确认有点击，那么根据鼠标的位置更新物件的位置。

```c#
using UnityEngine;
using UnityEngine.EventSystems;

public class UIScroll : MonoBehaviour, IPointerDownHandler, IPointerUpHandler {

    bool isClicked;
    Vector3 originalPosition;
    Vector3 off, oldPos;

    void Start () {
        isClicked = false;
        originalPosition = transform.position;
    }

    public void OnPointerDown(PointerEventData eventData) {
        oldPos = Input.mousePosition;
        oldPos.z = 10;
        oldPos = Camera.main.ScreenToWorldPoint(oldPos);
        isClicked = true;
    }

    public void OnPointerUp(PointerEventData eventData){
        isClicked = false;
    }

    void Update () {
        if (isClicked == true)
        {
            if (transform.position.x <= originalPosition.x  && transform.position.x >= -17) {
                //拖拽的位置
                Vector3 mousePos = Input.mousePosition;
                mousePos.z = 10;
                mousePos = Camera.main.ScreenToWorldPoint(mousePos);

                off = mousePos - oldPos;
                oldPos = mousePos;

                transform.position += new Vector3(off.x, 0, 0);
                //Debug.Log("Position.x: " + transform.position.x);
            }
            else if (transform.position.x > originalPosition.x) {
                transform.position = originalPosition;
            }
            else if (transform.position.x < -17) {
                transform.position = new Vector3(-17, 0, 90);
                Debug.Log("Entered edge");
            }
        }
    }
}
```
