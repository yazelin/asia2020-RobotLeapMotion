## KUKA 機器手臂與Leap Motion

### KUKA 機器手臂簡介
- KUKA [官網](https://www.kuka.com/) / [wiki](https://zh.wikipedia.org/wiki/%E5%BA%93%E5%8D%A1)

## 手臂基本介紹
### 六軸機器手臂
![Image](./img/RobotSystem.jpg)

### 軸向
- A1~A6 

![Image](./img/RobotAxis.jpg)

### 空間
- Base空間

![Image](./img/RobotCoordinateSystem.jpg)

- Tool空間

![Image](./img/Tool.jpg) 

### 線上模擬環境
- 我們把模擬環境放在網站上了
- 網址在這邊  [RobotSim WebPlayer](http://www.wtech.com.tw/robotsim/demo)
- 在模擬器中我們可以學到這些
- 座標系
  - WORLD
  - BASE
  - TOOL  
- 操作方式
  - XYZ ABC
  - AXIS
- 運動指令
  - PTP
  - LIN
  - CIRC(網頁版的模擬器中沒有) 
- 軸極限  
  - A1~A6
- 手臂程式執行方式
  - 先教點
  - 用指令讓手臂重現動作

### Leap Motion簡介
  - 使用擷取手掌姿態的裝置
  - [Leap Motion Controller](https://www.ultraleap.com/product/leap-motion-controller/)
  
## Leap Motion與機器手臂互動介紹
#### Unity+leapmotion與機器手臂
<iframe width="560" height="315"
src="./demo.mp4" 
frameborder="0" 
allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" 
allowfullscreen></iframe>

-   [其他參考影片1](https://www.facebook.com/wisetech.dakuo/videos/1212236958861791/)
-   [其他參考影片2](https://www.facebook.com/wisetech.dakuo/videos/1225804447505042/)

#### 硬體
- Robot KR6 R700-2
- Leapmotion
- PC
- 使用 USB 連結 Leapmotion 與 PC
- 使用 網路線 連結 PC 與 機器手臂
#### 軟體
- EKI KUKA網路通訊介面
- Unity 遊戲引擎
- Leapmotion SDK
#### 通訊方式
- EKI XML 格式
```
<Data><ActPos X="38.28448" Y="-4.016232" Z="-10.91361" A="0" B="0" C="0">0</ActPos></Data>
```
- 手臂端設定檔
```
<ETHERNETKRL>
	<CONFIGURATION>  
		<EXTERNAL>  
			<TYPE>Client</TYPE>  
		</EXTERNAL>  
		<INTERNAL>  
			<IP>192.168.1.147</IP>  
			<PORT>54600</PORT>  
			<ALIVE Set_Flag="1"/>  
		</INTERNAL>  
	</CONFIGURATION>  
	<RECEIVE>  
		<XML>  
			<ELEMENT Tag="Data/ActPos" Type="INT" Set_Flag="2" />  
			<ELEMENT Tag="Data/ActPos/@X" Type="REAL" />  
			<ELEMENT Tag="Data/ActPos/@Y" Type="REAL" />  
			<ELEMENT Tag="Data/ActPos/@Z" Type="REAL" />  
			<ELEMENT Tag="Data/ActPos/@A" Type="REAL" />  
			<ELEMENT Tag="Data/ActPos/@B" Type="REAL" />  
			<ELEMENT Tag="Data/ActPos/@C" Type="REAL" />  
		</XML>  
	</RECEIVE>  
	<SEND>  
		<XML>  
			<ELEMENT Tag="ROBOT/ACK" Type="STRING"/>  
		</XML>  
	</SEND>  
</ETHERNETKRL>
```
- 手臂端程式
```
DEF  LeapMotionExample ( )
   DECL Wt_Pos_Q_DATA_STRUC DATA
   DECL EKI_STATUS RET
   FRAME MOVETO
   
   GLOBAL INTERRUPT DECL 3 WHEN $STOPMESS==TRUE DO IR_STOPM ( )
   INTERRUPT ON 3 
   BAS (#INITMOV,0)
   
   GLOBAL INTERRUPT DECL 100 WHEN $FLAG[2] == TRUE DO Command_Processing()
   INTERRUPT ON 100
   
   RET = EKI_Init(LEAP_SERVER[])
   RET = EKI_Open(LEAP_SERVER[])
   
   INTERRUPT ON 100
   
   MOVETO = {X 0,Y 0,Z 0,A 0,B 0,C 0}
   
   
   BAS(#TOOL ,1)
   BAS(#BASE ,1)
   PTP $POS_ACT
   PTP {X 0,Y 0,Z 0,A 0,B 0,C 0}
   
   Wt_Pos_Q_Init ( )
   
   WHILE TRUE
      IF NOT Wt_Pos_Q_IsEmpty() THEN
         DATA = Wt_Pos_Q_Dequeue()
         MOVETO.X = DATA.DATA_X
         MOVETO.Y = DATA.DATA_Y
         MOVETO.Z = DATA.DATA_Z
         PTP MOVETO C_DIS         
      ENDIF
   ENDWHILE
   
END

DEF Command_Processing()
   DECL EKI_STATUS RET
   DECL Wt_Pos_Q_DATA_STRUC DATA
   
   REAL _X ,_Y ,_Z 
   
   _X = 0
   _Y = 200
   _Z = 0
   
   RET = EKI_GetREAL(LEAP_SERVER[] ,"Data/ActPos/@X" ,_X)
   RET = EKI_GetREAL(LEAP_SERVER[] ,"Data/ActPos/@Y" ,_Y)
   RET = EKI_GetREAL(LEAP_SERVER[] ,"Data/ActPos/@Z" ,_Z)
   RET = EKI_ClearBuffer(LEAP_SERVER[],"Data/ActPos")
   
   
   DATA.DATA_X = _X
   DATA.DATA_Y = _Y
   DATA.DATA_Z = _Z
   IF NOT Wt_Pos_Q_IsFull() THEN
      Wt_Pos_Q_Enqueue(DATA)
   ENDIF
   
   $FLAG[2] = FALSE
   RET = EKI_Send(LEAP_SERVER[] ,"e")
END
```
- PC端程式
```
using UnityEngine;
using System.Collections;
using System;
using System.Net;
using System.Net.Sockets;
using System.IO;

public class LeapRobot : MonoBehaviour
{

    public string ip = "192.168.1.147";
    public int port = 54600;
    private TcpClient tcp;
    private StreamReader sr;
    private StreamWriter sw;
    private bool isUpdating;
	private GameObject go;
	
    void Start()
    {

        isUpdating = false;
        try
        {
            tcp = new TcpClient(ip, port); // 1.設定 IP:Port 2.連線至伺服器(Robot)
            sr = new StreamReader(tcp.GetStream());
            sw = new StreamWriter(tcp.GetStream());

        }
        catch (SocketException ex)
        {
            Debug.Log(ex);
        }
        catch (Exception ex)
        {
            Debug.Log(ex);
        }
    }

    // Update is called once per frame
    void Update()
    {
        GameObject go = GameObject.Find("HandModels/RigidRoundHand_R/index/bone3");
        if (go)
        {
            if (!isUpdating)
            {
                isUpdating = true;
                Debug.Log("waitFinish");
                StartCoroutine(waitFinish(go.transform.position));
            }
        }
    }

    IEnumerator waitFinish(Vector3 unityPos)
    {
        Vector3 pos;
        string str;
        pos.x = -(float)(unityPos.z * 1000);
        pos.y = (float)(unityPos.x * 1000);
        pos.z = (float)(unityPos.y * 1000);
        //限制範圍
        if (-100 < pos.x && pos.x < 100 && -200 < pos.y && pos.y < 200 && -100 < pos.z && pos.z < 100)
        {
            str = "<Data><ActPos X=\"" + pos.x + "\" Y=\"" + pos.y + "\" Z=\"" + pos.z + "\" A=\"0\" B=\"0\" C=\"0\">0</ActPos></Data>";
            Debug.Log(str);
            sw.WriteLine(str); // 將資料寫入緩衝
            sw.Flush(); // 刷新緩衝並將資料傳到伺服器

            while (sr.Read() != 'e')
            {
                yield return new WaitForSeconds(0.001f);
            }

        }

        isUpdating = false;
    }
}

```
## 互動體驗
- 通訊方式會影響手臂動作流暢度
- 使用EKI方式 [參考連結](http://forum.wtech.com.tw/viewtopic.php?f=2&t=38)
- 使用RSI方式 [參考連結](http://forum.wtech.com.tw/viewtopic.php?f=2&t=158)



<!--stackedit_data:
eyJoaXN0b3J5IjpbMTAyODk2OTUyMywtMTg1NjQxOTkyMywxMj
I2ODY3OTk2LC0xMjgzOTc5MzgzLDQzOTAzOTYyNywyMDM3MjMy
MTc2LC0xMDMwMTYzMDg4LC0xOTM1MjQ3NDA1LDUyNTc1MTkwMS
wtMTg5NDYwOTI1NCwtODYwNTQyMzc3LC00MzIwNDIwNTFdfQ==

-->