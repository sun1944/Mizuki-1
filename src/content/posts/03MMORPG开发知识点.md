---
title: MMORPG游戏开发学习记录
published: 2025-09-15
description: "MMORPG游戏开发学习中的重点知识记录与讲解。"
image: "https://typorytp.oss-cn-hangzhou.aliyuncs.com/titlemmo.png"
tags: ["网络", "MMO", "Socket","UGUI","C#"]
category: Unity
draft: false
---


# MMORPG游戏开发学习记录

## 网络同步之状态同步

状态同步就是把玩家的信息发送给服务端，然后服务端计算返回结果同步给其他客户端，对于关键逻辑判定就是在服务端上进行的，这样保证了同步状态的权威性和数据安全性。





## 战斗系统



## UI系统设计

### 界面设计实现技巧

#### 加载界面

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.SceneManagement;
using UnityEngine.UI;

public class LoadPanel : BasePanel
{

    //加载进度条
    private Slider processViewSli;
    
    public override void OnInit()
    {

        processViewSli = transform.Find("Slider").GetComponent<Slider>();
        base.OnInit();

    }

    public override void OnShow(params object[] objs)
    {
        base.OnShow(objs);
        StartCoroutine(StartLoading(1));
        
        
    }

    private IEnumerator StartLoading(int sceneIndex)
    {
        int displayProgress = 0;
        int toProgress = 0;
        AsyncOperation ao = SceneManager.LoadSceneAsync(sceneIndex);
        ao.allowSceneActivation = false;
        while (ao.progress < 0.9f)
        {
            toProgress = (int)ao.progress * 100;
            while (displayProgress<toProgress)
            {
                displayProgress++;
                SetLoadingPercentValue(displayProgress);
                yield return new WaitForEndOfFrame();
            }
        }

        toProgress = 100;
        while (displayProgress < toProgress)
        {
            displayProgress++;
            SetLoadingPercentValue(displayProgress);
            yield return new WaitForEndOfFrame();
        }

        yield return new WaitForSeconds(1);
        ao.allowSceneActivation = true;
    }

    private void SetLoadingPercentValue(int value)
    {
        processViewSli.value = value / 100;
    }

}

~~~

## 位置判定

### 可导航点位判定

实现思路：简单来说，通过在一个三维网格中，系统地生成大量的候选点，然后使用 `NavMesh.SamplePosition` 方法逐个“询问”NavMesh系统：“这个点附近有可以行走的表面吗？”。

首先根据场景中的模型碰撞体生成遍历的点位，如果这个点位和导航平面有碰撞，则记为1，表示可到达，反之记为0。然后将判定的结果存储在一个三维列表中，之后再转成一维数组序列化为json格式，存储到文件中，提供给服务端用于判断。

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;
using UnityEngine.AI;
using JetBrains.Annotations;
using System.IO;



public class MapDataTool 
{
    [MenuItem("MapDataGenerateTool/Generate NavmeshData")]
    public static void GenerateNavmeshData()
    {
        GameObject sceneMap = GameObject.Find("Plane");
        
        float step = 0.8f;
        MeshCollider mc = sceneMap.GetComponent<MeshCollider>();

        //用于观看效果
        GameObject mapCube = GameObject.Find("MapCube");
        GameObject map = new GameObject("Map");//方便管理
        
        //通过三个维度遍历去生成可以行走的区域
        for (float x = mc.bounds.min.x; x < mc.bounds.max.x; x += step)
        {
            for (float z = mc.bounds.min.z; z < mc.bounds.max.z; z += step)
            {
                for (float y = mc.bounds.max.y; y < mc.bounds.max.y + 20; y += step)
                {
                    Vector3 pos = new Vector3(x, y, z);
                    NavMeshHit hit;
                    //生成的网格与检测点在0.5范围内命中
                    if (NavMesh.SamplePosition(pos, out hit, 0.5f, NavMesh.AllAreas))
                    {
                        GameObject g = GameObject.Instantiate(mapCube, map.transform);
                        g.name = x + "," + y + "," + z;
                        g.transform.position = pos;
                        g.transform.localScale = Vector3.one * 0.9f;
                    }
                }
            }
        }

        int sizeX = Mathf.RoundToInt(mc.bounds.size.x / step);
        int sizeY = Mathf.RoundToInt(mc.bounds.size.y + 20 - mc.bounds.size.y / step);
        int sizeZ = Mathf.RoundToInt(mc.bounds.size.z / step);
        MapData mapData = new MapData();
        mapData.xLength = sizeX;
        mapData.yLength = sizeY;
        mapData.zLength = sizeZ;
        mapData.boundX = mc.bounds.size.x;
        mapData.boundY = mc.bounds.size.y;
        mapData.boundZ = mc.bounds.size.z;
        int[,,] mData = new int[sizeX, sizeY, sizeZ];
        for (float x = mc.bounds.min.x; x < mc.bounds.max.x; x += step)
        {
            int indexX = Mathf.RoundToInt((x - mc.bounds.min.x) / step);
            for (float z = mc.bounds.min.z; z < mc.bounds.max.z; z += step)
            { 
                int indexZ = Mathf.RoundToInt((z - mc.bounds.min.z) / step);
                for (float y = mc.bounds.max.y; y < mc.bounds.max.y + 20; y += step)
                {
                    int indexY = Mathf.RoundToInt((y - mc.bounds.min.y) / step);
                    Vector3 pos = new Vector3(x, y, z);
                    NavMeshHit hit;
                    if (indexX >= sizeX)
                    {
                        indexX = sizeX - 1;
                    }
                    if (indexY >= sizeY)
                    {
                        indexY = sizeY - 1;
                    }
                    if (indexZ >= sizeZ)
                    {
                        indexZ = sizeZ - 1;
                    }

                    if (NavMesh.SamplePosition(pos, out hit, 0.5f, NavMesh.AllAreas))
                    {
                        GameObject g = GameObject.Instantiate(mapCube, map.transform);
                        g.name = indexX + "," + indexY + "," + indexZ;
                        g.transform.position = pos;
                        g.transform.localScale = Vector3.one * 0.5f;
                        mData[indexX, indexY, indexZ] = 1;
                        for (int i = indexY; i < indexY + 6; i++)
                        {
                            if (i >= sizeY)
                            {
                                break;
                            }
                            mData[indexX, i, indexZ] = 1;
                        }
                    }
                    
                }
            }
        }
        mapData.datas = Convert3DTo1D(mData, mapData);
        string json = JsonUtility.ToJson(mapData);
        string filePath = Application.streamingAssetsPath + "/Map.json";
        File.WriteAllText(filePath, json);

    }

    private static int[] Convert3DTo1D(int[,,] array3D, MapData mapData)
    {
        int xLength = mapData.xLength;
        int yLength = mapData.yLength;
        int zLength = mapData.zLength;
        
        mapData.xLength = xLength;
        mapData.yLength = yLength;
        mapData.zLength = zLength;
        
        int[] data = new int[xLength * yLength * zLength];
        int index = 0;
        for (int x = 0; x < xLength; x++)
        {
            for (int y = 0; y < yLength; y++)
            {
                for (int z = 0; z < zLength; z++)
                {
                    data[index] = array3D[x, y, z];
                    index++;
                }
            }
        }
        return data;
        
    }
    
    
}


[System.Serializable]
public class MapData
{
    public int[] datas;
    public int xLength;
    public int yLength;
    public int zLength;
    public float boundX;
    public float boundY;
    public float boundZ;
    public float gridStep;
    public int AOIMapXLength;
    public int AOIMapYLength;
}

~~~



## AOI算法

### 使用目的

一个联机游戏，服务器要同步客户端之间的信息。对于一个多人游戏来说，如果服务器内需要同步的客户端很多，那么计算量会非常巨大，给服务器造成很大压力。例如玩家A释放了一个技能，如果不加处理，理论上来说服务器会将玩家A释放一个技能的事件发送给全部客户端，即使他们没有任何联系或者距离太远而互不相干。

这时我们就需要使用AOI算法，服务器端不会向与A玩家互不相关的客户端发送同步信息，与A玩家互不相关的客户端将不会收到任何关于A的相关信息，这样就减轻了服务器带宽的压力。

### 实现思路

**距离法**

首先是最直观的距离法，简单来说就是服务器遍历玩家A与其他玩家之间的距离，只有在符合条件的玩家，服务端才会向玩家A发送同步信息并显示。

**格子法**

格子法则是把地图拆分成独立的网格，在同一个网格区域内的玩家才会相互同步，不在同一区域则不同步。但如果玩家处于格子边缘的位置，即使与相邻的格子很近也无法显示同步相关信息。

**九宫格法**

九宫格法则是格子法的优化实现，同样是将游戏地图拆分成二维的网格，同步玩家所在的格子以及周围的八个格子玩家的信息，在玩家跨格子移动时更新九宫格位置，控制相关玩家信息的同步。

### 具体实现方法

#### 服务端

在服务端的Player类里定义两个变量用于存储所在格子的索引。

~~~C#
public int AOIGridX;
public int AOIGridY;
~~~

在gamemanager类中定义如下变量和方法，用于读取地图数据以及初始化地图信息。AOIPlayerArray用于存储每个格子中所有的玩家列表,玩家刚进入游戏时设置一下所在格子索引。

~~~cs
	//AOI
    private float AOIGridStep;
    private List<Player>[,] AOIPlayerArray;
    private int AOIMapXLength;
    private int AOIMapYLength;
~~~

~~~cs
	public void ReadMapDatas()
    {
        string jsonPath = "Map.json";
        string filePath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory,jsonPath);
        string jsonStr = File.ReadAllText(filePath);
        MapData mapData = JsonConvert.DeserializeObject<MapData>(jsonStr);
        //MapData mapData= jss.Deserialize<MapData>(jsonStr);
        xLength = mapData.xLength;
        yLength = mapData.yLength;
        zLength = mapData.zLength;
        gridStep = mapData.gridStep;
        boundX = mapData.boundX;
        boundY = mapData.boundY;
        boundZ = mapData.boundZ;
        mapDatas = Convert1DTo3D(mapData.datas,xLength,yLength,zLength);
        //JudgePointInMap(-2.403f, 0.3868f, -12.43f);//101,0,108 -2.403 0.3868 -12.43

        //AOI
        AOIGridStep = gridStep * 15;
        AOIMapXLength = mapData.AOIMapXLength;
        AOIMapYLength = mapData.AOIMapYLength;
        AOIPlayerArray = new List<Player>[AOIMapXLength,AOIMapYLength];
        for (int x = 0; x < AOIMapXLength; x++)
        {
            for (int y = 0; y < AOIMapYLength; y++)
            {
                AOIPlayerArray[x, y] = new List<Player>();
            }
        }

    }
~~~

~~~cs
	/// <summary>
    /// 设置初始位置所在格子
    /// </summary>
    /// <param name="player"></param>
    public void SetDefaultGrid(Player player)
    {
        player.AOIGridX = (int)Math.Round((player.psd.x - boundX) / AOIGridStep);
        player.AOIGridY = (int)Math.Round((player.psd.z - boundZ) / AOIGridStep);
        AOIPlayerArray[player.AOIGridX, player.AOIGridY].Add(player);
    }
~~~

这是AOI的核心方法,在同步人物信息的协议事件中调用,具体思路是先获取玩家最新所在格子索引，获取老格子索引，当所在格子更新时，将新的九个格子的索引存储在一个列表里，将旧的九个格子存储在一个列表里，因为玩家每跨过一个格子，不是所有的九个格子都要更新索引，实际上只有其中的一排或一列离开，并新增一排或者一列。所以为了进行优化，需要把重复的格子剔除。之后要发送两次协议，需要通知旧格子区域玩家离开，通知新格子区域玩家进入。

**中心通知四周**

**目的：** 告诉**其他人**，“我”（移动的玩家）来了或者走了。

**解决了什么问题：**

当玩家离开旧格子时，他需要告诉原来九宫格内的所有玩家：“我走了，你们可以把我隐藏/销毁了”。（广播离开）

当玩家进入新格子时，他需要告诉新九宫格内的所有玩家：“我来了，你们要把我创建/显示出来”。（广播进入）

**如果只做这一步会怎样？**

移动的玩家成功通知了所有人，其他玩家都能正确地看到或隐藏他。

但是！ 移动的玩家自己的客户端没有得到任何信息。他不知道在这个新的格子里，原来都有哪些玩家。结果就是：其他玩家都能看见他，但他却看不见别人，他屏幕上是空的。这显然是不对的。

**四周通知中心 **

**目的：** 告诉**“我”**（移动的玩家），你新进入/离开的区域里都有谁。

**解决了什么问题：**

当玩家离开旧格子时，旧九宫格内的玩家会把自己的列表发给中心玩家（移动者），意思是：“我们这些人在你的旧视野里，现在你走了，你应该把我们隐藏掉”。（四周通知中心 - 离开）

当玩家进入新格子时，新九宫格内的玩家会把自己的列表发给中心玩家（移动者），意思是：“我们这些人在你的新视野里，你现在应该把我们显示出来”。（四周通知中心 - 进入）

**如果只做这一步会怎样？**

移动的玩家成功地知道了周围的所有玩家，他的客户端可以正确显示所有人。

但是！ 他并没有通知别人自己的到来。结果就是：他能看见别人，但别人却看不见他，他像一个“幽灵”。这也是不对的。



~~~cs
/// <summary>
    /// 判断当前玩家在哪个格子里
    /// </summary>
    /// <param name="player"></param>
    public void AOIDetect(Player player,PTSyncCharacter pt)
    {
        //当前玩家所在的新格子索引
        int x =(int)Math.Round((player.psd.x - boundX) / AOIGridStep);
        int y= (int)Math.Round((player.psd.z - boundZ) / AOIGridStep);
        //当前玩家所在的老格子索引
        int pX = player.AOIGridX;
        int pY = player.AOIGridY;
        if (pX!=x||pY!=y)
        {
            List<Grid> leaveGridsList = new List<Grid>();
            List<Grid> enterGridsList = new List<Grid>();
            //把当前离开的九宫格（九个格子）添加进离开列表里
            AddGridsToList(pX,pY,leaveGridsList);
            //把当前进入的九宫格（九个格子）添加进进入列表里
            AddGridsToList(x, y, enterGridsList);
            //把两个九宫格之间重复的部分剔除
            RemoveBothGridsInLists(ref leaveGridsList,ref enterGridsList);
            //在当前格子区域广播离开的玩家信息，让他销毁或隐藏
            PTSyncEnterOrLeaveAOI p = new PTSyncEnterOrLeaveAOI();
            p.enterAOI = false;
            p.pd = player.psd.PSDtoPD();
            ///Console.WriteLine("当前检测到的玩家是："+player.pd.id);
            //AOIBoardcastPTMessage(p,pX,pY);
            //中心通知四周
            AOIBoardcastPTMessage(p,leaveGridsList);
            //四周通知中心
            AOIBoardcastPTMessageToCenter(leaveGridsList,false,pX,pY);
            //离开当前格子
            AOIPlayerArray[pX, pY].Remove(player);
            //进入新格子
            AOIPlayerArray[x, y].Add(player);
            player.AOIGridX = x;
            player.AOIGridY = y;
            //在当前格子区域广播进入的玩家信息，让他生成或显示
            PTSyncEnterOrLeaveAOI pp = new PTSyncEnterOrLeaveAOI();
            pp.enterAOI = true;
            pp.pd = player.psd.PSDtoPD();
            //AOIBoardcastPTMessage(pp, x,y);
            //中心通知四周
            AOIBoardcastPTMessage(pp,enterGridsList);
            //四周通知中心
            AOIBoardcastPTMessageToCenter(enterGridsList, true, x, y);
        }
        else
        {
            AOIBoardcastPTMessage(pt, x, y);
        }
    }
~~~

协议定义

~~~cs
public class PTSyncEnterOrLeaveAOI : PTBase
{
	//服务器发
	public PTSyncEnterOrLeaveAOI() { protoName = "PTSyncEnterOrLeaveAOI"; }
	public PlayerData pd;
	public List<PlayerData> otherPlayerCDList;
	public bool enterAOI;
}

~~~

以下是相关方法的实现

~~~cs
/// <summary>
    /// 把对应的九宫格添加进对应列表里
    /// </summary>
    /// <param name="gridX">中心格子索引</param>
    /// <param name="gridY"></param>
    /// <param name="list"></param>
    private void AddGridsToList(int gridX, int gridY, List<Grid> list)
    {
        for (int x = gridX - 1; x < gridX + 2; x++)
        {
            if (gridX + 2 >= AOIPlayerArray.GetLength(0))
            {
                continue;
            }
            for (int y = gridY - 1; y < gridY + 2; y++)
            {
                if (gridY + 2 >= AOIPlayerArray.GetLength(1))
                {
                    continue;
                }
                list.Add(new Grid() { x = x, y = y });
            }
        }
    }
    /// <summary>
    /// 剔除两个九宫格相交的格子部分
    /// </summary>
    private void RemoveBothGridsInLists(ref List<Grid> leaveGridsList,ref List<Grid> enterGridsList)
    {
        //leaveTemp = leaveGridsList.Where(item=>!enterGridsList.Contains(item)).ToList();
        //enterTemp = enterGridsList.Where(item=>!leaveGridsList.Contains(item)).ToList();
        List<Grid> leaveTemp = leaveGridsList.ToList().Except(enterGridsList).ToList();
        //enterTemp = enterGridsList.ToList().Except(leaveGridsList).ToList();
        leaveGridsList.Clear();
        //enterGridsList.Clear();
        leaveGridsList.AddRange(leaveTemp);
        //enterGridsList.AddRange(enterTemp);
        //leaveTemp.Clear();
        //enterTemp.Clear();
    }

    /// <summary>
    /// 九宫格法AOI消息广播(中心格子向其他格子广播)
    /// </summary>
    /// <param name="pt"></param>
    /// <param name="gridX"></param>
    /// <param name="gridY"></param>
    public void AOIBoardcastPTMessage(PTBase pt,int gridX,int gridY)
    {
        for (int x = gridX-1; x < gridX+2; x++)
        {
            if (gridX + 2 >= AOIPlayerArray.GetLength(0))
            {
                continue;
            }
            for (int y = gridY-1; y < gridY+2; y++)
            {
                if (gridY + 2 >= AOIPlayerArray.GetLength(1))
                {
                    continue;
                }
                for (int i = 0; i < AOIPlayerArray[x, y].Count; i++)
                {
                    AOIPlayerArray[x, y][i].Send(pt);
                }
            }
        }
    }
    /// <summary>
    /// 九宫格法AOI消息广播(中心格子向其他格子广播)
    /// </summary>
    /// <param name="pt"></param>
    /// <param name="list">其他格子索引列表</param>
    public void AOIBoardcastPTMessage(PTBase pt, List<Grid> list)
    {
        for (int i = 0; i < list.Count; i++)
        {
            int x = list[i].x;
            int y = list[i].y;
            for (int j = 0; j < AOIPlayerArray[x,y].Count; j++)
            {
                AOIPlayerArray[x, y][j].Send(pt);
            }
        }
    }

    /// <summary>
    /// 九宫格法AOI消息广播(其他格子向中心格子广播)
    /// </summary>
    /// <param name="list">其他格子</param>
    /// <param name="centerX">中心格子索引</param>
    /// <param name="centerY"></param>
    public void AOIBoardcastPTMessageToCenter(List<Grid> list,bool enterAOI,int centerX,int centerY)
    {
        PTSyncEnterOrLeaveAOI p = new PTSyncEnterOrLeaveAOI();
        p.enterAOI = enterAOI;
        p.otherPlayerCDList = new List<PlayerData>();
        p.pd = new PlayerData();
        for (int i = 0; i < list.Count; i++)
        {
            //获取格子
            int x = list[i].x;
            int y = list[i].y;
            //遍历格子中当前的所有玩家
            for (int j = 0; j < AOIPlayerArray[x, y].Count; j++)
            {
                p.otherPlayerCDList.Add(AOIPlayerArray[x, y][j].psd.PSDtoPD());
            }
        }
        //遍历中心格子当前的所有玩家
        for (int j = 0; j < AOIPlayerArray[centerX, centerY].Count; j++)
        {
            AOIPlayerArray[centerX, centerY][j].Send(p);
        }
    }
    /// <summary>
    /// 获取当前九宫格内的所有玩家
    /// </summary>
    /// <param name="gridX"></param>
    /// <param name="gridY"></param>
    /// <returns></returns>
    public List<Player> PlayersInAOIGrid(int gridX,int gridY)
    {
        List<Player> list = new List<Player>();
        for (int x = gridX - 1; x < gridX + 2; x++)
        {
            if (gridX + 2 >= AOIPlayerArray.GetLength(0))
            {
                continue;
            }
            for (int y = gridY - 1; y < gridY + 2; y++)
            {
                if (gridY + 2 >= AOIPlayerArray.GetLength(1))
                {
                    continue;
                }
                list.AddRange(AOIPlayerArray[x, y]);
            }
        }
        return list;
    }
~~~

#### 客户端

客户端收到协议之后设置目标列表对象的位置以及相关信息，如果列表没有该对象则生成一个。

~~~cs
	/// <summary>
    /// 有人物进入或退出AOI
    /// </summary>
    /// <param name="pt"></param>
    public void OnPTSyncEnterOrLeaveAOI(PTBase pt)
    {
        PTSyncEnterOrLeaveAOI p = (PTSyncEnterOrLeaveAOI)pt;
        if (p.otherPlayerCDList.Count>0)
        {
            for (int i = 0; i < p.otherPlayerCDList.Count; i++)
            {
                HandleAOIUpdateInfo(p, p.otherPlayerCDList[i]);
            }
        }
        else
        {
            HandleAOIUpdateInfo(p,p.pd);
        }
    }
    private void HandleAOIUpdateInfo(PTSyncEnterOrLeaveAOI p,PlayerData pd)
    {
        if (pd.id != currentPSD.id)//只处理其他人物
        {
            if (syncPMCDict.ContainsKey(pd.id))
            {
                syncPMCDict[pd.id].gameObject.SetActive(p.enterAOI);
                //先更新状态信息
                if (p.enterAOI)
                {
                    Vector3 pos = new Vector3(pd.x, pd.y, pd.z);
                    Vector3 rot = new Vector3(pd.ex, pd.ey, pd.ez);
                    syncPMCDict[pd.id].ImmediateUpdateSyncPosAndRot(pos, rot, pd.characterState);
                }
            }
            else
            {
                if (pd.id==""|| !p.enterAOI)
                {
                    return;
                }
                //第一次进入我们当前客户端玩家的视野范围时
                //生成
                //生成新进入玩家的角色游戏物体
                GameObject newGo = GameObject.Instantiate(GameResSystem.GetRes<GameObject>("Prefabs/Character/SyncPlayer"),
                    new Vector3(pd.x, pd.y, pd.z), Quaternion.Euler(pd.ex, pd.ey, pd.ez));
                newGo.name = pd.id;
                SyncPMCtrl spmc = newGo.GetComponent<SyncPMCtrl>();
                spmc.isAI = pd.isAI;
                spmc.InitDressState(pd);
                AddNewPlayerData(pd, spmc);
            }
        }
    }
~~~

## AI玩家

### 实现思路

AI玩家的行为不再是接收客户端消息进行控制的了，所以AI的逻辑直接放在服务端上广播给附近客户端就可以了。

首先创建AIPlayer类继承自Player类，设置AI玩家的行为逻辑，例如攻击、巡逻等功能，在PlayerManager类中定义生成AI玩家的方法，用于控制AI玩家的显示与生成，AI玩家定时向附近玩家发送同步消息。

首先先在构造函数中初始化所需信息，开启定时器，用于定时向相关客户端发送NPC的同步消息 。

~~~cs
public AIPlayer(ClientObject clientObject, PlayerSaveData playerData,List<GridIndex> pplist) : base(clientObject)
{
    psd = new PlayerSaveData();
    id = psd.id = playerData.id;
    psd.level = playerData.level;
    psd.gender = playerData.gender;
    psd.isAI = true;
    psd.role = playerData.role;
    patrolPath.AddRange(pplist);
    GridIndex fgi = patrolPath[0];
    targetPatrolPath = initPos = GameManager.Instance.GetPointPosInMap(fgi.x,fgi.y,fgi.z);
    psd.rd = PlayerManager.Instance.GetBasicRoleAttributeValueData(psd.role)+ 
        PlayerManager.Instance.GetGrowthRoleAttributeValueData(psd.role)*2;
    //GameManager.Instance.SetDefaultGrid(this);
    psd.x = targetPatrolPath.x;
    psd.y = targetPatrolPath.y;
    psd.z = targetPatrolPath.z;
    psd.hp = psd.rd.HP;
    psd.mana = psd.rd.mana;
    //Console.WriteLine(psd.rd.strength);
    sendPtTimer.Elapsed += SendPTToPlayer;
    sendPtTimer.Start();
}
~~~

AI行为逻辑控制，无需多说。

~~~cs
	/// <summary>
    /// 定时向当前AOI区域内的玩家发送自身状态信息
    /// </summary>
    /// <param name="sender"></param>
    /// <param name="e"></param>
    public void SendPTToPlayer(object sender, ElapsedEventArgs e)
    {
        if (psd.characterState==CHARACTERSTATE.DEAD)
        {
            //NPC已死
            sendPtTimer.Stop();
            return;
        }
        PTSyncCharacter ptsc = new PTSyncCharacter();
        //NPC位置
        Vector3 npcPos = new Vector3(psd.x,psd.y,psd.z);
        if (target != null)
        {
            //有攻击目标
            Vector3 tPos = new Vector3(target.psd.x, target.psd.y, target.psd.z);
            if (Vector3.Distance(npcPos, tPos) >=psd.rd.attackRange)
            {
                //没有到达攻击范围
                psd.characterState = CHARACTERSTATE.MOVE;
                //朝目标移动
                npcPos = npcPos + (tPos - npcPos).normalized * sendPTTimeval / 1000 * psd.rd.moveSpeed;
            }
            else
            {
                //到达攻击范围
                if (target.psd.hp > 0)
                {
                    //目标未死
                    psd.characterState = CHARACTERSTATE.BATTLE;
                }
                else
                {
                    //目标已阵亡
                    target = null;
                    //重置当前目标并通知当前AOI区域的其他玩家
                    PTSyncSetChoiceTarget p = new PTSyncSetChoiceTarget();
                    p.pID = psd.id;
                    p.tID = null;
                    GameManager.Instance.AOIBoardcastPTMessage(p, AOIGridX, AOIGridY);
                }
            }
        }
        else
        {
            //无攻击目标
            psd.characterState = CHARACTERSTATE.MOVE;
            //我们离开始追击的位置是不是大于1米（代表我们不是平时的巡逻状态，
            //而是去做了其他事情需要返回）
            if (Vector3.Distance(npcPos, initPos) > 1)
            {
                //返回初始位置
                npcPos += (initPos - npcPos).normalized * sendPTTimeval / 1000 * psd.rd.moveSpeed;
            }
            else
            {
                //正常巡逻状态
                npcPos= Patrol(npcPos);
                initPos = npcPos;
                //回到正常巡逻状态时，血量需要回满
                if (psd.hp != psd.rd.HP)
                {
                    psd.hp = psd.rd.HP;
                    hpHasChanged = true;
                }
            }
        }
        psd.x = npcPos.x;
        psd.y = npcPos.y;
        psd.z = npcPos.z;
        //psd.characterState = CHARACTERSTATE.MOVE;
        //攻击判定
        JudgeAttackEvent(false);
        //受击判定
        JudgeHitEvent();
        //目标搜索判定
        JudgeTargetEvent(npcPos);
        ptsc.cd = psd.PSDtoPD().PDToCD();
        GameManager.Instance.AOIDetect(this, ptsc);
    }
~~~

相关方法实现如下

~~~cs
/// <summary>
    /// 搜索当前AOI区域内的玩家以寻找攻击目标
    /// </summary>
    /// <param name="npcPos"></param>
    public void JudgeTargetEvent(Vector3 npcPos)
    {
        if (target!=null)
        {
            //有目标不搜索
            return;
        }
        List<Player> l= GameManager.Instance.PlayersInAOIGrid(AOIGridX,AOIGridY);
        float minDis = float.MaxValue;
        //遍历当前AOI区域内所有的玩家，找到离我们最近的作为攻击对象
        for (int i = 0; i < l.Count; i++)
        {
            Player p = l[i];
            if (p.psd.id==psd.id||p.psd.characterState==CHARACTERSTATE.DEAD)
            {
                continue;
            }
            Vector3 pPos = new Vector3(p.psd.x,p.psd.y,p.psd.z);
            float dis= Vector3.Distance(npcPos,pPos);
            if (dis<=chaseRange)
            {
                if (dis<minDis)
                {
                    minDis = dis;
                    target = p;
                }
            }
        }
        //如果搜索到目标，则发送设置目标的协议告诉所有AOI区域内的玩家，并开始攻击
        if (target!=null)
        {
            PTSyncSetChoiceTarget p = new PTSyncSetChoiceTarget();
            p.pID = psd.id;
            p.tID = target.psd.id;
            GameManager.Instance.AOIBoardcastPTMessage(p,AOIGridX,AOIGridY);
            StartAttack();
        }
    }

    public Vector3 Patrol(Vector3 npcPos)
    {
        if (Vector3.Distance(targetPatrolPath,npcPos)<=0.5f)
        {
            pathIndex++;
            if (pathIndex>=patrolPath.Count)
            {
                pathIndex = 0;
            }
            GridIndex fgi = patrolPath[pathIndex];
            targetPatrolPath = GameManager.Instance.GetPointPosInMap(fgi.x,fgi.y,fgi.z);
        }
        else
        {
           npcPos+= (targetPatrolPath - npcPos).normalized * psd.rd.moveSpeed * sendPTTimeval / 1000;
        }
        return npcPos;
    }
~~~

