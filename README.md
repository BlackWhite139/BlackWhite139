#include "StdAfx.h"
#ifdef _STANDALONE
#ifndef __linux
#include <winsock2.h>
#include <malloc.h>
#else
#include <unistd.h>
#include <netdb.h>
#include <sys/time.h>
#endif
#include "KWin32.h"
#include "package.h"
#endif

#include "CRC32.h"
#ifdef _STANDALONE
#include "KCore.h"
#include "KPlayerSet.h"
#include "CRC32.c"
#endif

#ifndef WIN32
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/ioctl.h>
#include <netinet/in.h>
#include <net/if.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <string.h>
#endif

#define TIME_FARM 200
#define TIME_FARM_H 100
#define _SERVER

#ifdef _STANDALONE
//	#ifdef WIN32
//		#define FAILED(x) ((x) != S_OK)
//	#else
	#define FAILED(x) ((x) <= 0)
//	#endif
#endif

#ifndef _STANDALONE
#include <crtdbg.h>
#include <objbase.h>
#include <initguid.h>
#include "Library.h"
#include "inoutmac.h"
#endif

#include <iostream>
//#include <iostream.h>
//#include <strstream>

#include "S3DBInterface.h"
#include "KProtocolDef.h"
#include "KProtocol.h"
#include "KRelayProtocol.h"
#include "KTongProtocol.h"

#include "KSOServer.h"
//#include <afxmt.h>
//#pragma comment(linker,"/STACK:102400000,1024000")


#ifndef _STANDALONE
using OnlineGameLib::Win32::CBuffer;
using OnlineGameLib::Win32::CLibrary;
using OnlineGameLib::Win32::CCriticalSection;
CLibrary g_theHeavenLibrary( "heaven.dll" );
CLibrary g_theRainbowLibrary( "rainbow.dll" );
OnlineGameLib::Win32::CUsesWinsock g_SocketInit;
CCriticalSection g_csFlow;
#else
  CMutex g_mutexFlow;
#endif

CPackager			m_theRecv;
CPackager			m_theSend;

#define				GAME_FPS		18

#define				GFDHFGRT		"3503492A3C679101A2AAA177C1D9050A"
#define				FDGFGYKL        "U0ASZS3ZCAQGxrqeNGAB06Mhc8dG3dcA"
#define				RIUHJFHI        "F36B291419E617501B2F6769BD3A9A1" 	//2018071505

using namespace std;

//const int KSwordOnLineSever::m_snMaxPlayerCount = 500;
//const int KSwordOnLineSever::m_snPrecision = 50;
//const int KSwordOnLineSever::m_snMaxBuffer=1024;
//int KSwordOnLineSever::m_snBufferSize=1024*64; //16 »ò  200 
//³õÊ¼»¯
KSwordOnLineSever g_pSOServer;

enum PLAYER_EXCHANGE_STATUS
{
	enumExchangeBegin = 0,
	enumExchangeSearchingWay,
	enumExchangeWaitForGameSvrRespone,
	enumExchangeCleaning,
};


int	g_nTongPCSize[defTONG_PROTOCOL_CLIENT_NUM] = 
{
	-1,										// enumS2C_TONG_CREATE_SUCCESS
	sizeof(STONG_CREATE_FAIL_SYNC),			// enumS2C_TONG_CREATE_FAIL
	sizeof(STONG_ADD_MEMBER_SUCCESS_SYNC),	// enumS2C_TONG_ADD_MEMBER_SUCCESS
	sizeof(STONG_ADD_MEMBER_FAIL_SYNC),		// enumS2C_TONG_ADD_MEMBER_FAIL
	-1,										// enumS2C_TONG_HEAD_INFO
	-1,										// enumS2C_TONG_MANAGER_INFO
	-1,										// enumS2C_TONG_MEMBER_INFO
	sizeof(STONG_BE_INSTATED_SYNC),			// enumS2C_TONG_BE_INSTATED
	sizeof(STONG_INSTATE_SYNC),				// enumS2C_TONG_INSTATE
	sizeof(STONG_KICK_SYNC),				// enumS2C_TONG_KICK
	sizeof(STONG_BE_KICKED_SYNC),			// enumS2C_TONG_BE_KICKED
	sizeof(STONG_LEAVE_SYNC),				// enumS2C_TONG_LEAVE
	sizeof(STONG_CHECK_GET_MASTER_POWER_SYNC),	// enumS2C_TONG_CHECK_CHANGE_MASTER_POWER
	sizeof(STONG_CHANGE_MASTER_FAIL_SYNC),	// enumS2C_TONG_CHANGE_MASTER_FAIL
	sizeof(STONG_CHANGE_AS_SYNC),			// enumS2C_TONG_CHANGE_AS
	sizeof(STONG_CHANGE_MASTER_SYNC),		// enumS2C_TONG_CHANGE_MASTER
	sizeof(STONG_LOGIN_DATA_SYNC),			// enumS2C_TONG_LOGIN_DATA
	sizeof(STONG_MONEY_SYNC),
	sizeof(STONG_MONEY_SYNC),
	sizeof(STONG_MONEY_SYNC),
};



#ifndef _STANDALONE
typedef HRESULT ( __stdcall * pfnCreateClientInterface )( 
			REFIID riid, 
			void **ppv 
		);

typedef HRESULT ( __stdcall * pfnCreateServerInterface )(
			REFIID	riid,
			void	**ppv
		);
#endif

#ifndef _STANDALONE
void __stdcall ServerEventNotify(
#else
void ServerEventNotify(								 
#endif
			LPVOID lpParam,
			const unsigned long &ulnID,
			const unsigned long &ulnEventType )
{

#ifndef _STANDALONE
	CCriticalSection::Owner locker(g_csFlow);
#else
	g_mutexFlow.Lock();
#endif

	switch(ulnEventType)
	{
	case enumClientConnectCreate: //ÉèÖÃÍøÂçÁ´½Ó×´Ì¬
		g_pSOServer.SetNetStatus(ulnID, enumNetConnected);
		//printf("----------ÍøÂç(%d) ×´Ì¬Á¬½Ó----------\n",ulnID);
		break;
	case enumClientConnectClose:  //Àë¿ªÓÎÏ·Ê±µ÷ÓÃ ÉèÖÃ¶Ï¿ª×´Ì¬
		g_pSOServer.SetNetStatus(ulnID, enumNetUnconnect,true);
	    //printf("----------ÍøÂç(%d) ×´Ì¬¶Ï¿ª----------\n",ulnID);
		break;
	}

#ifdef _STANDALONE
	g_mutexFlow.Unlock();
#endif

}


#ifndef _STANDALONE
void __stdcall GatewayClientEventNotify(
#else
void GatewayClientEventNotify(
#endif
			LPVOID lpParam,
			const unsigned long &ulnEventType )
{
	switch( ulnEventType )
	{
	case enumServerConnectCreate:
		g_pSOServer.m_mapClientNet[GATWSTATE]=TRUE;
		break;
	case enumServerConnectClose:
		//g_pSOServer.SetS3RelayRunStatus(FALSE,5);
		g_pSOServer.m_mapClientNet[GATWSTATE]=FALSE;
		printf("---------GateWay Lost----------\n");
		//g_SOServer.SetRunningStatus(FALSE);  //ÉèÖÃGSÍ£Ö¹ÔËÐÐ
		break;
	}
}

#ifndef _STANDALONE
void __stdcall ChatClientEventNotify(
#else
void ChatClientEventNotify(
#endif
			LPVOID	lpParam,
			const unsigned long &ulnEventType )
{
	switch( ulnEventType )
	{
	case enumServerConnectCreate:
		g_pSOServer.m_mapClientNet[CHATSTATE]=TRUE;
		break;
	case enumServerConnectClose:
		//g_pSOServer.m_ChatState = FALSE;
		//g_pSOServer.SetS3RelayRunStatus(FALSE,1);
		g_pSOServer.m_mapClientNet[CHATSTATE]=FALSE;
		printf("-------Chat Disconnect-----\n");
		break;
	}
}

#ifndef _STANDALONE
void __stdcall TongClientEventNotify(
#else
void TongClientEventNotify(
#endif
	LPVOID	lpParam,const unsigned long &ulnEventType)
{
	switch( ulnEventType )
	{
	case enumServerConnectCreate:	// Á¬½Ó½¨Á¢Ê±ºòµÄ´¦Àí
		g_pSOServer.m_mapClientNet[TONGSTATE]=TRUE;
		break;
	case enumServerConnectClose:	// Á¬½Ó¶Ï¿ªÊ±ºòµÄ´¦Àí
		//g_pSOServer.m_TongState = FALSE;
		g_pSOServer.m_mapClientNet[TONGSTATE]=FALSE;
		//g_pSOServer.SetS3RelayRunStatus(FALSE,2);
		printf("------Tèng Disconnect------\n");
		break;
	}
}

#ifndef _STANDALONE
void __stdcall DatabaseClientEventNotify(
#else
void DatabaseClientEventNotify(
#endif
			LPVOID lpParam,
			const unsigned long &ulnEventType)
{
	switch(ulnEventType)
	{
	case enumServerConnectCreate:
		g_pSOServer.m_mapClientNet[DATASTATE]=TRUE;
		break;
	case enumServerConnectClose:
		//g_pSOServer.SetS3RelayRunStatus(FALSE,4);
		g_pSOServer.m_mapClientNet[DATASTATE]=FALSE;
		printf("---------DataBase Lost-----------\n");
		//g_pSOServer.SetRunningStatus(FALSE);//ÉèÖÃGSÍ£Ö¹ÔËÐÐ
		break;
	}
}

#ifndef _STANDALONE
void __stdcall TransferClientEventNotify(
#else
void TransferClientEventNotify(
#endif
			LPVOID lpParam,
			const unsigned long &ulnEventType )
{
	switch( ulnEventType )
	{
	case enumServerConnectCreate:
		g_pSOServer.m_mapClientNet[TRANSTATE]=TRUE;
		break;
	case enumServerConnectClose:
//		g_SOServer.SetRunningStatus(FALSE);
		g_pSOServer.m_mapClientNet[TRANSTATE]=FALSE;
		//g_pSOServer.SetS3RelayRunStatus(FALSE,3);
		//g_pSOServer.m_TranState = FALSE;
		break;
	}
}

KSwordOnLineSever::KSwordOnLineSever()
{
	m_nGameLoop = 0;
	m_nGameDay  = 0;
	m_nCurGameLoop=0;
	m_bIsRunning = TRUE;
	m_nServerPort = 56766;      //µÇÂ½ÓÎÏ··þÎñÆ÷µÄ¶Ë¿Ú
	m_nGatewayPort = 56732;     //bishop ÕËºÅµÇÂ½IP
	m_nDatabasePort = 5001;
	m_nTransferPort = 5003;
	m_nChatPort	= 5004;
	m_nTongPort	= 5005;
	m_snBufferSize=1024*16;
	m_snMaxBuffer=1024;
	ZeroMemory(m_szGatewayIP, sizeof(m_szGatewayIP));
	ZeroMemory(m_szDatabaseIP, sizeof(m_szDatabaseIP));
	ZeroMemory(m_szTransferIP, sizeof(m_szTransferIP));
	ZeroMemory(m_szChatIP, sizeof(m_szChatIP));
	ZeroMemory(m_szTongIP, sizeof(m_szTongIP));
	m_pServer = NULL;
	m_pGatewayClient = NULL;
	m_pDatabaseClient = NULL;
	m_pTransferClient = NULL;
	m_pChatClient = NULL;
	m_pTongClient = NULL;
	m_pCoreServerShell = NULL;
	m_pGameStatus = NULL;
	m_IsWm=TRUE;
	inGo=FALSE;

    m_ChatState = FALSE;
    m_TongState = FALSE;
    m_TranState = FALSE;
	m_pDataState = FALSE;
    m_GatewayState = FALSE;
	nCurWindHwnd = 0;
	ZeroMemory(m_CurTime,sizeof(m_CurTime));

	m_mapClientNet.clear();
}

KSwordOnLineSever::~KSwordOnLineSever()
{
}

//ÉèÖÃ·þÎñÆ÷¶Ë¿Ú
BOOL KSwordOnLineSever::SetServerPort()
{
	g_SetRootPath(NULL);
	g_SetFilePath("\\");
	KIniFile iniFile;
	if (!iniFile.Load("ServerCfg.ini"))
	{
		MessageBox(0,"GSÆô¶¯Ê§°Ü","GS:",MB_OK); 
	}

	iniFile.GetInteger("GameServer", "Port",56766, &m_nServerPort);
/*
#ifdef WIN32
	iniFile.GetInteger("Overload", "MaxPlayer", 450, &m_nMaxPlayerCount);
	iniFile.GetInteger("Overload", "Precision", 10, &m_nPrecision);

#else
	iniFile.GetInteger("Overload", "MaxPlayer", 1000, &m_nMaxPlayerCount);
	iniFile.GetInteger("Overload", "Precision", 10, &m_nPrecision);
#endif */
	iniFile.Clear();

  if (m_pServer)
  {
	if (FAILED(m_pServer->OpenService(INADDR_ANY,m_nServerPort)))
	{//´ò¿ª·þÎñÆ÷¶Ë¿Ú
		//cout << "Can't OPEN server port\n" << endl;
		//MessageBox(0,"GSÆô¶¯»ñÈ¡·þÎñÆ÷IPAÊ§°Ü","GS:",MB_OK);
	}
	else
	{
		printf("-----¶þ´Î¿ª·Å¶Ë¿Ú³É¹¦:%d----\n",m_nServerPort);

	}
		//printf("-------------·þÎñÆ÷¶Ë¿ÚA:%d-------------\n",m_nServerPort);
	/*	tagGameSvrInfo ni;			
			ni.cProtocol = c2s_updategameserverinfo;
			
			ni.nIPAddr_Internet = m_dwInternetIp;
			ni.nIPAddr_Intraner = m_dwIntranetIp;
			ni.nPort            = m_nServerPort;
			ni.wCapability      = m_nMaxPlayerCount;  //×î´óÈËÊýÏÞÖÆ
			m_pGatewayClient->SendPackToServer((const void *)&ni, sizeof(tagGameSvrInfo));*/
  }
	return TRUE;
}

void KSwordOnLineSever::ErrOut()
{
	//cout<<".......·þÎñÆ÷ÒÑ¾­Ìø³ög_pSOServer.Breathe()Ñ­»·......"<<endl;
	printf(".......·þÎñÆ÷ÒÑ¾­ÍË³ö»î¶¯...... \n");

}

//È¡ÓàÊý
/*DWORD TakeRemainder(DWORD a,DWORD b)
{
	DWORD nRet = 0;
	//	DWORD nYuShu=0;
	__asm
	{
		mov eax,a
			mov ecx,b
			xor edx,edx
			idiv ecx
			mov nRet,edx
			//mov nYuShu,edx
	}
	return nRet;
}

//È¡ÉÌ
DWORD TakeTrader(DWORD a,DWORD b)
{
	DWORD nRet = 0;
	//DWORD nYuShu=0;
	__asm
	{
		mov eax,a
			mov ecx,b
			xor edx,edx
			idiv ecx
			mov nRet,eax
			//mov nYuShu,edx
	}
	return nRet;
} */

int	KSwordOnLineSever::FindFree()
{
	return m_FreeIdxNetStatus.GetNext(0);
}

int KSwordOnLineSever::FindSame(DWORD dwID)
{
	int nUseIdx = 0;
	
	nUseIdx = m_UseIdxNetStatus.GetNext(0);
	while(nUseIdx)
	{
		if (m_pGameStatus[nUseIdx].nNetidx == dwID)
			return nUseIdx;
		nUseIdx = m_UseIdxNetStatus.GetNext(nUseIdx);
	}
	return 0;
}

int	KSwordOnLineSever::GetFirstNetIdx()
{
	m_nListCurIdx = m_UseIdxNetStatus.GetNext(0);
	return m_nListCurIdx;
}

int	KSwordOnLineSever::GetNextNetIdx()
{
	if (!m_nListCurIdx)
		return 0;
	m_nListCurIdx = m_UseIdxNetStatus.GetNext(m_nListCurIdx);
	return m_nListCurIdx;
}

void KSwordOnLineSever::ClearNetStatus()
{
	int nUsedIndex = m_UseIdxNetStatus.GetNext(0);
	
	while (nUsedIndex!= 0)
	{
		//Remove(nUsedIndex);
		m_FreeIdxNetStatus.Insert(nUsedIndex);
		m_UseIdxNetStatus.Remove(nUsedIndex);
		nUsedIndex = m_UseIdxNetStatus.GetNext(0);
	}
}
//-------------------------¼à¿ØÏß³Ì
//¹¤×÷Ïß³Ì
DWORD WINAPI ThreadProcessGetGatway( void *pParam )
{
	IClient *pcServer = (IClient *)pParam;
	if (pcServer)
	{
	  cout << " - Íø¹Ø·þÎñÆ÷¼à¿ØÖÐ.!" << endl;
	  DWORD g_nServiceLoop = 0;
	  while(true)
	  {
		if(!g_pSOServer.GetClientNet(GATWSTATE))
		{//ÒÑ¾­¶Ï¿ª ¾Í¿ªÊ¼ÖØÁ¬
			if  (g_nServiceLoop>=TIME_FARM && (g_nServiceLoop%TIME_FARM)==0)
			     g_pSOServer.SetClientConnetNet(1);
		}

		//try
		//{ 
			g_pSOServer.PlayerLogoutGateway();           //Íø¹Ø¶Ï¿ª Ñ­»·¼ì²â ÊÇ·ñÍË³öÓÎÏ·µÄ
		//} 
		/*catch (...)
		{ 
			printf("Core PlayerLogoutGateway !\n");
			g_pSOServer.m_pGatewayClient->Shutdown();	 //¹Ø±ÕÁ´½Ó È»ºóÖØÁ¬
			g_pSOServer.m_mapClientNet[GATWSTATE] = FALSE;
			g_pSOServer.m_GatewayState = FALSE;
		}*/

		if (++g_nServiceLoop & 0x80000000)
		{
			g_nServiceLoop = 0;
		}
		if (g_nServiceLoop & 0x1)
		{
			::Sleep(10);
		}
	  }
	}
	
	return 0L;
}

DWORD WINAPI ThreadProcessDataClient( void *pParam )
{
	IClient *pcServer = (IClient *)pParam;
	if (pcServer)
	{
	   cout << " - Êý¾Ý¿â·þÎñÆ÷¼à¿ØÖÐ.!" << endl;
	   DWORD g_nServiceLoop = 0;
	   while(true)
	   {
		if(!g_pSOServer.GetClientNet(DATASTATE))
		{//ÒÑ¾­¶Ï¿ª ¾Í¿ªÊ¼ÖØÁ¬
			if  (g_nServiceLoop>=TIME_FARM && (g_nServiceLoop%TIME_FARM)==0)
				g_pSOServer.SetClientConnetNet(2);
		}
		if (++g_nServiceLoop & 0x80000000)
		{
			g_nServiceLoop = 0;
		}
		if (g_nServiceLoop & 0x1)
		{
			::Sleep(10);
		}
	   }
	}
	return 0L;
}

DWORD WINAPI ThreadProcessChatClient( void *pParam )
{
	IClient *pcServer = (IClient *)pParam;
	if (pcServer)
	{
	  cout << " - ÁÄÌì·þÎñÆ÷¼à¿ØÖÐ.!" << endl;
	  DWORD g_nServiceLoop = 0;
	  while(true)
	  {
		if(!g_pSOServer.GetClientNet(CHATSTATE))
		{//ÒÑ¾­¶Ï¿ª ¾Í¿ªÊ¼ÖØÁ¬
			if  (g_nServiceLoop>=TIME_FARM && (g_nServiceLoop%TIME_FARM)==0)
				g_pSOServer.SetClientConnetNet(3);
		}

		if (++g_nServiceLoop & 0x80000000)
		{
			g_nServiceLoop = 0;
		}
		if (g_nServiceLoop & 0x1)
		{
			::Sleep(10);
		}

	  }
	}
	return 0L;
}

DWORD WINAPI ThreadProcessTongClient( void *pParam )
{
	IClient *pcServer = (IClient *)pParam;
	if (pcServer)
	{
		cout << " - °ï»á·þÎñÆ÷¼à¿ØÖÐ.!" << endl;
	  DWORD g_nServiceLoop = 0;
	  while(true)
	  {
		if(!g_pSOServer.GetClientNet(TONGSTATE))
		{//ÒÑ¾­¶Ï¿ª ¾Í¿ªÊ¼ÖØÁ¬
			if  (g_nServiceLoop>=TIME_FARM && (g_nServiceLoop%TIME_FARM)==0)
				g_pSOServer.SetClientConnetNet(4);
		}
		if (++g_nServiceLoop & 0x80000000)
		{
			g_nServiceLoop = 0;
		}
		if (g_nServiceLoop & 0x1)
		{
			::Sleep(10);
		}
	}
	}
	return 0L;
}

DWORD WINAPI ThreadProcessTranClient( void *pParam )
{
	IClient *pcServer = (IClient *)pParam;
	#define	MAX_STAT_QUERY_TIME 800*3600           //Ã¿ËÄ¸öÐ¡Ê±²éÑ¯Ò»´Î

	if (pcServer)
	{
	   cout << " - ¿ç·þ·þÎñÆ÷¼à¿ØÖÐ.!" << endl;
	   DWORD g_nServiceLoop = 0;
	   while(true)
	   {
		if(!g_pSOServer.GetClientNet(TRANSTATE))
		{//ÒÑ¾­¶Ï¿ª ¾Í¿ªÊ¼ÖØÁ¬
			if  (g_nServiceLoop>=TIME_FARM && (g_nServiceLoop%TIME_FARM)==0)
				g_pSOServer.SetClientConnetNet(5);
		}
		//ÅÅÃû²éÑ¯
		if  (g_nServiceLoop>=MAX_STAT_QUERY_TIME && (g_nServiceLoop%MAX_STAT_QUERY_TIME)==0)
			g_pSOServer.SetClientConnetNet(8);

		if  (g_nServiceLoop>=6 && (g_nServiceLoop%6)==0) //Ã¿°ëÃë
		//LONGLONG nMes = 1000;
		//if (nMes * g_pSOServer.m_nGameLoop <= g_pSOServer.m_Timer.GetElapse()*GAME_FPS) //1Ãë18Ö¡  Ê±¼äÒç³ö 
		     g_pSOServer.SetClientConnetNet(9);
		/*else
		{
			printf("-------[Ê±¼äÅÐ¶ÏÓÐÎó,ÔÝÍ£ 1 Ãë]--------- \n");
			Sleep(1);  //Í£Ö¹Ò»Ãë	usleep
		}*/
		
		if (++g_nServiceLoop & 0x80000000)
		{
			g_nServiceLoop = 0;
		}
		if (g_nServiceLoop & 0x1)
		{
			::Sleep(10);
		}
	   }
	}
	return 0L;
}

DWORD WINAPI ThreadSendClientData( void *pParam )
{
	IServer* pcMianServer =(IServer *)pParam;
	if (pcMianServer)
	{
		cout << " - ·¢°üÎñ·þÎñÆ÷¼à¿ØÖÐ.!" << endl;
		DWORD g_nServiceLoop = 0;
		while(true)
		{
			g_pSOServer.SendCilentPackData();

			if (++g_nServiceLoop & 0x80000000)
			{
				g_nServiceLoop = 0;
			}
			if (g_nServiceLoop & 0x1)
			{
				::Sleep(10);
			}
		}
	}
	return 0L;
}

DWORD WINAPI ThreadSaveDataTask( void *pParam )
{
	IClient *pcServer = (IClient *)pParam;
	if (pcServer)
	{
		cout << " - ´æµµÈÎÎñ·þÎñÆ÷¼à¿ØÖÐ.!" << endl;
		DWORD g_nServiceLoop = 0;
		while(true)
		{
			//ÈÎÎñ ´æµµ
			if  (g_nServiceLoop>=TIME_FARM && (g_nServiceLoop%TIME_FARM)==0)
				g_pSOServer.SetClientConnetNet(6);
			//ÅÄÂôÐÐÍ¬²½
			if  (g_nServiceLoop>=TIME_FARM && (g_nServiceLoop%(300*TIME_FARM)) ==0)
				g_pSOServer.SetClientConnetNet(7);

			if (++g_nServiceLoop & 0x80000000)
			{
				g_nServiceLoop = 0;
			}
			if (g_nServiceLoop & 0x1)
			{
				::Sleep(10);
			}
		}
	}
	return 0L;
}


BOOL KSwordOnLineSever::InitServer()
{
	/*
	 * Initialize class member
	 */

	m_bIsRunning = TRUE;
	g_SetRootPath(NULL);
	g_SetFilePath("\\");
	KIniFile iniFile;
	if (!iniFile.Load("ServerCfg.ini"))
	{
		//MessageBox(0,"GSÆô¶¯Ê§°Ü","GS:",MB_OK);
        return FALSE;
	}

	iniFile.GetInteger("GameServer", "Port",56766,&m_nServerPort);
	extern int g_nPort;
	if (g_nPort)//Æô¶¯¶Ë¿Ú
		m_nServerPort = g_nPort;

	iniFile.GetString("Gateway", "Ip", "127.0.0.1", m_szGatewayIP,sizeof(m_szGatewayIP));
	iniFile.GetInteger("Gateway", "Port", 56732, &m_nGatewayPort);  //ÕËºÅµÇÂ½¶Ë¿Ú
	iniFile.GetString("Database", "Ip", "127.0.0.1", m_szDatabaseIP,sizeof(m_szDatabaseIP));
	iniFile.GetInteger("Database", "Port", 5001, &m_nDatabasePort);
	iniFile.GetString("Transfer", "Ip", "127.0.0.1", m_szTransferIP,sizeof(m_szTransferIP));
	iniFile.GetInteger("Transfer", "Port", 5003, &m_nTransferPort);
	iniFile.GetString("Chat", "Ip", "127.0.0.1", m_szChatIP,sizeof(m_szChatIP));
	iniFile.GetInteger("Chat", "Port", 5004, &m_nChatPort);
	iniFile.GetString("Tong", "Ip", "127.0.0.1", m_szTongIP,sizeof(m_szTongIP));
	iniFile.GetInteger("Tong", "Port", 5005, &m_nTongPort);

#ifdef WIN32
	iniFile.GetInteger("Overload", "MaxPlayer", 500, &m_nMaxPlayerCount);
	iniFile.GetInteger("Overload", "Precision", 10, &m_nPrecision);
	//iniFile.GetInteger("Overload", "BufferSize", 65536, &m_snBufferSize);
	iniFile.GetInteger("Overload", "MaxGameStaus", 5000, &m_nMaxGameStaus);
	iniFile.GetInteger("Overload", "MaxClienLoop", 50, &m_nMaxClienLoop);
	
#else
	iniFile.GetInteger("Overload", "MaxPlayer", 1000, &m_nMaxPlayerCount);
	iniFile.GetInteger("Overload", "Precision", 10, &m_nPrecision);
	//iniFile.GetInteger("Overload", "BufferSize", 65536, &m_snBufferSize);
	iniFile.GetInteger("Overload", "MaxGameStaus", 5000, &m_nMaxGameStaus);
	iniFile.GetInteger("Overload", "MaxClienLoop", 50, &m_nMaxClienLoop);
#endif

    iniFile.GetInteger("Overload", "SerVerNo",1,&m_SerVerNo); ///ÓÃÓÚ¶àÇøµÄ·þÎñÆ÷±àºÅ
	char m_szTransIP[16]={0};
	iniFile.GetInteger("TransServer", "TransSerVerNo",1,&mTransSerVerNo);
	iniFile.GetString("TransServer", "Ip", "127.0.0.1", m_szTransIP,sizeof(m_szTransIP));
	TransDwIp = inet_addr(m_szTransIP);
	iniFile.GetInteger("TransServer", "Port",0,&TransPort);
	iniFile.GetInteger("BufferSize","gsBufferSize",16*1024,&m_snBufferSize);
	iniFile.GetInteger("BufferSize","errorcount",50,&m_errorCount);
	iniFile.GetInteger("BufferSize","pingcount",20,&m_pingCount);
	
	
	//if (m_nMaxGameStaus <=0 || m_nMaxGameStaus>MAX_PLAYER )
	m_nMaxPlayerCount = MAX_PLAYER;
    m_nMaxGameStaus   = MAX_PLAYER;
	//if (m_nMaxGameStaus>MAX_PLAYER)
	//	m_nMaxGameStaus = MAX_PLAYER;
	//m_nMaxPlayer    = m_nMaxPlayerCount;
	//m_nMaxGameStaus = m_nMaxPlayerCount;
	//if  (m_nMaxPlayer<=0 || m_nMaxPlayer<MAX_PLAYER)
	m_nMaxPlayer      = MAX_PLAYER;

	if  (m_nMaxGameStaus>m_nMaxPlayer)
		m_nMaxGameStaus = m_nMaxPlayer;

	if (m_nMaxPlayer <= 0)
	{
		cout << "Maximal player number <= 0!" << endl;
//		return FALSE;
	}

	if (!m_pGameStatus)
	{
		m_pGameStatus = new GameStatus[m_nMaxGameStaus]; //m_nMaxPlayer	 200  0 ºÍ 200 ÎÞÐ§
	}
	int i;

	for (i = 0; i < m_nMaxGameStaus; ++i)
	{	
		m_pGameStatus[i].nNetStatus     = enumNetUnconnect;
		m_pGameStatus[i].nNetidx        = -1;
		m_pGameStatus[i].nReplyPingTime = 0;
		m_pGameStatus[i].nSendPingTime  = 0;
		m_pGameStatus[i].nPlayerIndex   = 0;
		m_pGameStatus[i].nGameStatus     = enumPlayerNone;
		m_pGameStatus[i].nExchangeStatus = enumExchangeCleaning;
		m_pGameStatus[i].nErrorLoopCount = 0;
		m_pGameStatus[i].nPingLoopCount  = 0;
	}

	// ÓÅ»¯²éÕÒ±í
	m_FreeIdxNetStatus.Init(m_nMaxGameStaus);
	m_UseIdxNetStatus.Init(m_nMaxGameStaus);
	
	// ¿ªÊ¼Ê±ËùÓÐµÄÊý×éÔªËØ¶¼Îª¿Õ
	for (i = m_nMaxGameStaus - 1; i > 0; i--)
	{//1--199
		m_FreeIdxNetStatus.Insert(i);
	}

//	strcpy(m_szGameSvrIP, OnlineGameLib::Win32::net_ntoa(dwIp));
	
	/*
	 * Open this server to player
	 */

#ifndef _STANDALONE
	pfnCreateServerInterface pFactroyFun = (pfnCreateServerInterface)(g_theHeavenLibrary.GetProcAddress("CreateInterface"));

	IServerFactory *pServerFactory = NULL;

	if (pFactroyFun && SUCCEEDED(pFactroyFun(IID_IServerFactory, reinterpret_cast<void **>(&pServerFactory ))))
	{
		pServerFactory->SetEnvironment(m_nMaxPlayerCount,m_nPrecision,m_nPrecision,m_snBufferSize,0); //m_snMaxBuffer  m_nMaxPlayerCount
		
		pServerFactory->CreateServerInterface(IID_IIOCPServer, reinterpret_cast<void **>(&m_pServer));
		
		pServerFactory->Release();
	}
	else
	{
		return FALSE;
	}
#else
	m_pServer = new IServer(m_nMaxPlayer,4,200*1024);//20,64  64KB
	
#endif
	 //printf("=======m_pServer ÄÚ´æ´óÐ¡ £º%d b============================\n",sizeof(IServer));
	if (!m_pServer)
	{
		cout << "Initialization failed! Don't find a correct heaven.dll" << endl;
//		return FALSE;
	}

//////////////////////////////////////////////////m_nMaxPlayerCount//////////////////////////////////
	//cout << " Gameserver Infomation\n Gax player    :   "<<m_nMaxPlayerCount<< "\n maximum player:   " <<m_nMaxPlayer <<"\n max npc       :   50.000\n max item      :   400.000\n max task      :   2.000"<< endl;
	printf("==========================================\n");
	printf("±¾·þÎñÆ÷¶ËÎªÉÌÒµÍøÂç°æ,½ö¹©½£ÏÀ°®ºÃÕßÓéÀÖ!\n");
	printf("ÐÞ¸´ÒÑÖªµÄBUG,ÁªÏµ:QQ:25557432(ºáµ¶ÏòÌìÐ¦)!\n");
    printf("====Í¬²½¹Ù·½×îÐÂ¿Í»§¶Ë,¹¦ÄÜÇ¿´ó===========\n");
    printf("===========================================\n");

	m_pServer->Startup();
    //×¢²áÐÅÏ¢¹ýÂËÆ÷º¯Êý
	m_pServer->RegisterMsgFilter(reinterpret_cast< void * >(m_pServer),ServerEventNotify);

	if (FAILED(m_pServer->OpenService(INADDR_ANY,m_nServerPort)))
	{//´ò¿ª·þÎñÆ÷¶Ë¿Ú
		//cout << "Can't get server ip" << endl;
		//MessageBox(0,"GSÆô¶¯´ò¿ª·þÎñÆ÷¶Ë¿ÚÊ§°Ü,¶Ë¿ÚÊÇ·ñ¸øÕ¼ÓÃÁË","GS:",MB_OK);
		cout<<"--Æô¶¯·þÎñÆ÷¶Ë¿ÚÊ§°Ü,¶Ë¿ÚÊÇ·ñ¸øÕ¼ÓÃÁË!--"<<endl;
		Sleep(5000);
		ExitProcess(0);  //ÍË³ö½ø³Ì
	}
    
	int inMacNum=0;
	     inMacNum=GetLocalIpAddress(&m_dwIntranetIp, &m_dwInternetIp);

	if (m_IsWm)
	{//Ö±½ÓµØÇøÍâÍøIP
		KIniFile iniFile;
		if (iniFile.Load("ServerCfg.ini"))
		{
			char strIP[32]={0};	
			iniFile.GetString("GameServer", "Ip","127.0.0.1",strIP,sizeof(strIP));	
			m_dwInternetIp=inet_addr(strIP);

		}
		iniFile.Clear();
	}

	if (inMacNum<0 || inMacNum>2)
	{
	    cout<<"--Íø¿¨¹ý¶à,±£³ÖÒ»ÐéÄâÍø¿¨,Ò»¹«ÍøÍø¿¨¼´¿É,ÆäÓàÇë½ûÓÃ!--"<<endl;
		Sleep(5000);
		ExitProcess(0);  //ÍË³ö½ø³Ì
		//MessageBox(0,"GSÆô¶¯»ñÈ¡·þÎñÆ÷IPBÊ§°Ü","GS:",MB_OK);
	}

	const unsigned char* pIpAddress=NULL;

	      pIpAddress=(const unsigned char*)&m_dwIntranetIp;

    char Address[128]={0}; //IPµØÖ·

	sprintf(Address, "%d.%d.%d.%d", pIpAddress[0], pIpAddress[1],pIpAddress[2], pIpAddress[3]);

	cout <<" ÐéÄâ IP   :   "<< Address << endl;

	pIpAddress=(const unsigned char*)&m_dwInternetIp;
	sprintf(Address, "%d.%d.%d.%d", pIpAddress[0], pIpAddress[1],pIpAddress[2], pIpAddress[3]);

	cout <<" ¹«Íø IP   :   "<< Address << endl;
//----------------------------µ¥»úÉèÖÃ	192.168.100.8 
      /*char ntnnn[64]={0},ndnnn[64]={0};
	   //PassChecksum nTss;
	   //nTss.SimplyDecryptPassword(ntnnn,GFDHFGRT);
	   JieMa_Decipher(GFDHFGRT,ntnnn);
	   sprintf(ndnnn,"%u",m_dwInternetIp);
	   if (!strstr(ndnnn,ntnnn))
	   //if (m_dwInternetIp!=atoi(ntnnn)
	   {
		   return FALSE;
	   }*/
//-----------------------------
#ifndef _STANDALONE
	IServer *pCloneServer = NULL;
	m_pServer->QueryInterface(IID_IIOCPServer,( void **)&pCloneServer);
#else 
	IServer *pCloneServer = m_pServer;
#endif
	/*
	 * Init GameWorld
	 */
	if (m_pCoreServerShell == NULL)
	{
		//m_pCoreServerShell = ::CoreGetServerShell();	   //Æô¶¯³õÊ¼ ÓÎÏ·Êý¾Ý
		m_pCoreServerShell = ::CoreGetServerShell("QGvVeEASisH1MHO0LX4MGAWewCXAAh3X");//µ¥»úÉèÖÃ
	}

	if (m_pCoreServerShell==NULL)
	{
		cout << "----------------ÓÎÏ·Êý¾Ý³õÊ¼»¯Ê§°Ü------------------" << endl;
		return FALSE;
	}

	m_pCoreServerShell->OperationRequest(SSOI_LAUNCH, (unsigned int)pCloneServer, 0);  //·¢ËÍ¿ªÊ¼ÓÎÏ·ÃüÁî

	/*
	 * - Ket noi den database
	 */
#ifndef _STANDALONE
	iniFile.GetInteger("BufferSize", "DataBuffer",131072,&m_snBufferSize);
	pfnCreateClientInterface pClientFactroyFun = (pfnCreateClientInterface)(g_theRainbowLibrary.GetProcAddress("CreateInterface"));
	IClientFactory *pClientFactory = NULL;

	if (pClientFactroyFun && SUCCEEDED(pClientFactroyFun(IID_IClientFactory,reinterpret_cast< void ** >(&pClientFactory))))
	{   
		pClientFactory->SetEnvironment(m_nMaxPlayer,m_nPrecision,m_snBufferSize);  //128*1024

		pClientFactory->CreateClientInterface( IID_IESClient, reinterpret_cast< void ** >(&m_pDatabaseClient));

		pClientFactory->Release();
	}
	else
	{
//		return FALSE;
       cout << "Failed to Create pClientFactory." << endl;
	}
#else
//	net_buffer = new ZBuffer(20240000, 400);
	m_pDatabaseClient = new IClient(6000*1024, 6000*1024);//6000
#endif
	//printf("=======m_pDatabaseClient ÄÚ´æ´óÐ¡ £º%d kb============================\n",sizeof(m_pDatabaseClient)/1024);
	if (!m_pDatabaseClient)
	{
		cout << "Initialization failed! Don't find a correct rainbow.dll" << endl;

//		return FALSE;
	}
	if (FAILED(m_pDatabaseClient->Startup()))
	{
		cout << "Can't not startup database client service!" << endl;
//		return FALSE;
	}
//	cout << " ___________________________________________"<<endl;
	cout << " Database IP   : " << m_szDatabaseIP << " - Port : " << m_nDatabasePort  <<endl;

	m_pDatabaseClient->RegisterMsgFilter(reinterpret_cast<void *>(m_pDatabaseClient),DatabaseClientEventNotify);
  	//printf("=======m_pDatabaseClient ÄÚ´æ´óÐ¡ £º%d kb============================\n",sizeof(m_pDatabaseClient)/1024);
	if (FAILED(m_pDatabaseClient->FsGameServerConnectTo(m_szDatabaseIP,m_nDatabasePort)))
	{//Á´½ÓÕâ¸ö¶Ë¿Ú
		m_pDataState = FALSE;
		cout << " - Êý¾Ý¿â·þÎñÆ÷Á´½Ó Ê§°Ü.!" << endl;
	}
	else
	{
		m_pDataState = TRUE;
		cout << " - Êý¾Ý¿â·þÎñÆ÷Á´½Ó OK..!" << endl;
	}

	IClient* pDataServer =m_pDatabaseClient;
	m_pCoreServerShell->OperationRequest(SSOI_DATA_INIT, (unsigned int)pDataServer, 0);  //·¢ËÍ¿ªÊ¼ÓÎÏ·ÃüÁî
	
	/*
	 * - Ket noi den transfer
	 */

#ifndef _STANDALONE
	iniFile.GetInteger("BufferSize", "TransBuffer",131072,&m_snBufferSize);
	pClientFactroyFun = (pfnCreateClientInterface)(g_theRainbowLibrary.GetProcAddress("CreateInterface"));
	pClientFactory = NULL;

	if (pClientFactroyFun && SUCCEEDED(pClientFactroyFun(IID_IClientFactory, reinterpret_cast<void **>(&pClientFactory))))
	{
		pClientFactory->SetEnvironment(m_nMaxPlayer,m_nPrecision,m_snBufferSize); //1024 * 512

		pClientFactory->CreateClientInterface(IID_IESClient, reinterpret_cast< void ** >(&m_pTransferClient));

		pClientFactory->Release();
	}
	else
	{
	//	return FALSE;
		cout << " - ´íÎóA SetEnvironment!" << endl;
	}
#else
	m_pTransferClient = new IClient(6000*1024, 6000*1024);
#endif
	//printf("=======m_pTransferClient ÄÚ´æ´óÐ¡ £º%d kb============================\n",sizeof(m_pTransferClient)/1024);
	if (!m_pTransferClient)
	{
		cout << "Initialization failed! Don't find a correct rainbow.dll" << endl;
		cout << " - ´íÎóB SetEnvironment!" << endl;
	}
		
	if (FAILED(m_pTransferClient->Startup()))
	{
		cout << "Can't not startup transfer client service!" << endl;
	}
	
	cout << " Transfer IP   : " << m_szTransferIP << " - Port : " << m_nTransferPort << endl;
	
	m_pTransferClient->RegisterMsgFilter( reinterpret_cast< void * >(m_pTransferClient),TransferClientEventNotify);
	//host server connet
	if (FAILED(m_pTransferClient->FsGameServerConnectTo(m_szTransferIP,m_nTransferPort)))
	{
		cout << "- Ket noi den transfer is failed!" << endl;
		/*if (m_pTransferClient)
		{
			m_pTransferClient->Shutdown();
			m_pTransferClient->Cleanup();
			m_pTransferClient->Release();
			m_pTransferClient = NULL;
		}*/
		m_TranState = FALSE;
	}
	else
	{	
		m_TranState = TRUE;
		cout << " - ¿ç·þ·þÎñÆ÷Á´½Ó OK..!" << endl;	
	}

	/*
	 * - Ket noi den chat
	 */
#ifndef _STANDALONE
	iniFile.GetInteger("BufferSize", "ChatBuffer",131072,&m_snBufferSize);
	pClientFactroyFun = (pfnCreateClientInterface)(g_theRainbowLibrary.GetProcAddress("CreateInterface"));

	pClientFactory = NULL;

	if ( pClientFactroyFun && SUCCEEDED(pClientFactroyFun(IID_IClientFactory,reinterpret_cast< void ** >(&pClientFactory))))
	{
		pClientFactory->SetEnvironment(m_nMaxPlayer,m_nPrecision,m_snBufferSize); //1024 * 1024 * 4

		pClientFactory->CreateClientInterface(IID_IESClient,reinterpret_cast< void ** >(&m_pChatClient));

		pClientFactory->Release();
	}
	else
	{
//		return FALSE;
			cout << " - ´íÎóD SetEnvironment!" << endl;
	}
#else
	m_pChatClient = new IClient(6000*1024, 6000*1024);
#endif
	//printf("=======m_pChatClient ÄÚ´æ´óÐ¡ £º%d kb============================\n",sizeof(m_pChatClient)/1024);
	if (!m_pChatClient)
	{
		cout << "Initialization failed! Don't find a correct rainbow.dll" << endl;

//		return FALSE;
	}
		
	if (FAILED(m_pChatClient->Startup()))
	{
		cout << "Can't not startup chat client service!" << endl;
//		return FALSE;
	}
	
	cout << " Chat IP       : " << m_szChatIP << " - Port : " << m_nChatPort << endl;
	
	m_pChatClient->RegisterMsgFilter( reinterpret_cast< void * >(m_pChatClient), ChatClientEventNotify );
	
	if (FAILED( m_pChatClient->FsGameServerConnectTo( m_szChatIP,m_nChatPort)))
	{
		cout << "- Ket noi den chat is failed!" << endl;
		m_ChatState = FALSE;
		/*if (m_pChatClient)
		{
			m_pChatClient->Shutdown();
			m_pChatClient->Cleanup();
			m_pChatClient->Release();
			m_pChatClient = NULL;
		}*/
	}
	else
	{	
		m_ChatState = TRUE;
		cout << " - ÁÄÌì·þÎñÆ÷Á´½Ó OK..!" << endl;
		//printf("  - ÁÄÌì·þÎñÆ÷Á´½Ó OK..!\n");
//		MessageBox(0," - ÁÄÌì·þÎñÆ÷Á´½Ó OK..!","GS:",MB_OK);
	}

//------------------------ - Ket noi den tong ----------------------------
#ifndef _STANDALONE
	iniFile.GetInteger("BufferSize", "TongBuffer",131072,&m_snBufferSize);
	pClientFactroyFun = (pfnCreateClientInterface)(g_theRainbowLibrary.GetProcAddress("CreateInterface"));
	pClientFactory = NULL;
	if ( pClientFactroyFun && SUCCEEDED( pClientFactroyFun(IID_IClientFactory, reinterpret_cast< void ** >(&pClientFactory))))
	{
		pClientFactory->SetEnvironment(m_nMaxPlayer,m_nPrecision,m_snBufferSize); //1024 * 1024 * 4
		pClientFactory->CreateClientInterface(IID_IESClient, reinterpret_cast<void **>(&m_pTongClient));
		pClientFactory->Release();
	}
	else
	{
//		return FALSE;
			cout << " - ´íÎóE SetEnvironment!" << endl;
	}
#else
	m_pTongClient = new IClient(6000*1024, 6000*1024);  //64*1024
#endif
	//printf("=======m_pTongClient ÄÚ´æ´óÐ¡ £º%d kb============================\n",sizeof(m_pTongClient)/1024);

	if ( !m_pTongClient )
	{
		cout << "Initialization failed! Don't find a correct rainbow.dll" << endl;
//		return FALSE;
	}
		
	if (FAILED(m_pTongClient->Startup()))
	{
		cout << "Can't not startup tong client service!" << endl;
//		return FALSE;
	}
	
	cout << " Tong IP       : " << m_szTongIP << " - Port : " << m_nTongPort << endl;
	
	m_pTongClient->RegisterMsgFilter( reinterpret_cast< void * >(m_pTongClient), TongClientEventNotify );
	
	if (FAILED( m_pTongClient->FsGameServerConnectTo(m_szTongIP, m_nTongPort)))
	{
		cout << " - Ket noi den tong is failed!" << endl;
		m_TongState = FALSE;
		/*if (m_pTongClient)
		{
			m_pTongClient->Shutdown();
			m_pTongClient->Cleanup();
			m_pTongClient->Release();
			m_pTongClient = NULL;
		}*/
	}
	else
	{	
		m_TongState = TRUE;
		cout << " - °ï»á·þÎñÆ÷Á´½Ó OK..!" << endl;
	//	printf("  - °ï»á·þÎñÆ÷Á´½Ó OK..!\n");
//		MessageBox(0," - °ï»á·þÎñÆ÷Á´½Ó OK..!","GS:",MB_OK);
	}
//------------------------ - Ket noi den tong end ----------------------------
	IClient* pTongServer =m_pTongClient;
	m_pCoreServerShell->OperationRequest(SSOI_TONG_INIT, (unsigned int)pTongServer, 0);  //·¢ËÍ¿ªÊ¼ÓÎÏ·ÃüÁî

	/*
	 * - Ket noi den gateway
	 */
#ifndef _STANDALONE
	iniFile.GetInteger("BufferSize", "GatewayBuffer",131072,&m_snBufferSize);
	pClientFactroyFun = (pfnCreateClientInterface )(g_theRainbowLibrary.GetProcAddress("CreateInterface"));

	pClientFactory = NULL;

	if (pClientFactroyFun && SUCCEEDED(pClientFactroyFun(IID_IClientFactory, reinterpret_cast<void **>(&pClientFactory))))
	{
		pClientFactory->SetEnvironment(m_nMaxPlayer,m_nPrecision,m_snBufferSize);  //1024 * 128

		pClientFactory->CreateClientInterface(IID_IESClient, reinterpret_cast<void ** >(&m_pGatewayClient) );

		pClientFactory->Release();
	}
	else
	{
//		return FALSE;
			cout << " - ´íÎóF SetEnvironment!" << endl;
	}
#else
	m_pGatewayClient = new IClient(6000*1024, 6000*1024);  //IClient(64*1024,64*1024)
#endif
	//printf("=======m_pGatewayClient ÄÚ´æ´óÐ¡ £º%d kb============================\n",sizeof(m_pGatewayClient)/1024);
	if (!m_pGatewayClient)
	{
		cout << "Initialization failed! Don't find a correct rainbow.dll" << endl;

//		return FALSE;
	}

	if (FAILED(m_pGatewayClient->Startup()))
	{//´´½¨Ïß³Ì
		cout << "Can't not startup gateway client service!" << endl;
//		return FALSE;
	}

	// HANDLE nHd = m_pGatewayClient->GetTheaidHdw();

	cout << " Gateway IP    : " << m_szGatewayIP << " - Port : " << m_nGatewayPort << endl;

	m_pGatewayClient->RegisterMsgFilter( reinterpret_cast< void * >( m_pGatewayClient ), GatewayClientEventNotify );

	 if (FAILED( m_pGatewayClient->FsGameServerConnectTo(m_szGatewayIP,m_nGatewayPort)))
	 { 
		m_GatewayState = FALSE;
		cout << "- Ket noi den gateway is failed!" << endl;
//		return FALSE;
	 } 
     else 
	 {
		 m_GatewayState = TRUE;
	     cout << " - ÕËºÅµÇÂ½Íø¹Ø·þÎñÆ÷Á´½Ó OK..!" << endl;
	 }
	 IClient* pGatewayServer =m_pGatewayClient;
	 m_pCoreServerShell->OperationRequest(SSOI_GATEWAYINIT, (unsigned int)pGatewayServer, 0);  //·¢ËÍ¿ªÊ¼ÓÎÏ·ÃüÁî
	cout << ">>>>>>>>>>>>>·þÎñÆ÷Æô¶¯ OK..<<<<<<<<<<<<<" << endl;	
	printf("==========================================\n");
	printf("±¾·þÎñÆ÷¶ËÎªÉÌÒµÍøÂç°æ,½ö¹©½£ÏÀ°®ºÃÕßÓéÀÖ!\n");
	printf("ÐÞ¸´ÒÑÖªµÄBUG,ÁªÏµ:QQ:25557432(ºáµ¶ÏòÌìÐ¦)!\n");
	printf("==========================================\n");
	
	/*
	UINT_PTR   SetTimer(   
	HWND   hWnd,   //   ´°¿Ú¾ä±ú   
	UINT_PTR   nIDEvent,   //   ¶¨Ê±Æ÷ID£¬¶à¸ö¶¨Ê±Æ÷Ê±£¬¿ÉÒÔÍ¨¹ý¸ÃIDÅÐ¶ÏÊÇÄÄ¸ö¶¨Ê±Æ÷   
	UINT   uElapse,   //   Ê±¼ä¼ä¸ô,µ¥Î»ÎªºÁÃë   
	TIMERPROC   lpTimerFunc   //   »Øµ÷º¯Êý   
	);   
	
	*/
	//SetTimer(g_mainwnd, 1,1000, NULL);
	//HMODULE hModule = ::GetModuleHandle(NULL);
	HMODULE hKernel32 = GetModuleHandle("kernel32");
	GetCurWindow = (PROCGETCONSOLEWINDOW)GetProcAddress(hKernel32,"GetConsoleWindow");
	nCurWindHwnd = GetCurWindow();

	printf("---µ±Ç°¾ä±ú£º%u----\n",nCurWindHwnd);

	if  (nCurWindHwnd>0)
	{
//		SetTimer(nCurWindHwnd, 1,1000, (TIMERPROC)(KSwordOnLineSever::TimerProc(nCurWindHwnd,0,1,1000)));
	   
	}

	m_Timer.Start();
    //m_Timer.Stop();
	m_nGameLoop = 0;
	m_nGameDay  = 0;
	m_nCurGameLoop=0;
	//------------------------------·¢ËÍ¸üÐÂS3¸÷ÖÖÊý¾Ý
	tagLeaveGame2 lg2;
	lg2.ProtocolFamily = pf_udataserveridx;
	lg2.ProtocolID     = relay_s2c_udateserver;	   // S3 Íæ¼ÒÀë¿ªÓÎÏ·
	lg2.cCmdType       = NORMAL_LEAVEGAME;
	lg2.nSelServer     = m_SerVerNo;
	if (m_pTransferClient)
		m_pTransferClient->SendPackToServer(&lg2,sizeof(tagLeaveGame2));

	//¿ªÊ¼Ïß³Ì¼à¿Ø
	//¿ªÊ¼¶ÁÈ¡ip
	//g_SetRootPath(NULL);
	//g_SetFilePath("\\");
	//KIniFile nClientInfo;
	//if (nClientInfo.Load("ServerCfg.ini"))
	//{
		/*iniFile.GetString("Gateway", "Ip", "127.0.0.1",g_szGatewayIP,sizeof(g_szGatewayIP));
		iniFile.GetInteger("Gateway", "Port", 56732, &g_nGatewayPort);  //ÕËºÅµÇÂ½¶Ë¿Ú
		iniFile.GetString("Database", "Ip", "127.0.0.1", g_szDatabaseIP,sizeof(g_szDatabaseIP));
		iniFile.GetInteger("Database", "Port", 5001, &g_nDatabasePort);
		iniFile.GetString("Transfer", "Ip", "127.0.0.1", g_szTransferIP,sizeof(g_szTransferIP));
		iniFile.GetInteger("Transfer", "Port", 5003, &g_nTransferPort);
		iniFile.GetString("Chat", "Ip", "127.0.0.1", g_szChatIP,sizeof(g_szChatIP));
		iniFile.GetInteger("Chat", "Port", 5004, &g_nChatPort);
		iniFile.GetString("Tong", "Ip", "127.0.0.1", g_szTongIP,sizeof(g_szTongIP));
		iniFile.GetInteger("Tong", "Port", 5005, &g_nTongPort);*/
		//iniFile.Clear();
//	}
	//¿ªÊ¼¼à¿ØÏß³Ì
	DWORD dwThreadID = 0L;
	static HANDLE g_hThread = NULL;
	//IClient* GetGatwayClient =NULL;
	//GetGatwayClient = g_pSOServer.GetGatwClient();
	g_hThread = ::CreateThread(NULL,0,ThreadProcessGetGatway,(void *)m_pGatewayClient,0,&dwThreadID);
	//IClient* DataClient=NULL;
	//DataClient= g_pSOServer.GetDataClient();
	g_hThread = ::CreateThread(NULL,0,ThreadProcessDataClient,(void *)m_pDatabaseClient,0,&dwThreadID);
	//IClient* Chatlient =NULL;
	//Chatlient = g_pSOServer.GetChatClient();
	g_hThread = ::CreateThread(NULL,0,ThreadProcessChatClient,(void *)m_pChatClient,0,&dwThreadID);
	//IClient* Tonglient =NULL;
	//Tonglient = g_pSOServer.GetTongClient();
	g_hThread = ::CreateThread(NULL,0,ThreadProcessTongClient,(void *)m_pTongClient,0,&dwThreadID);
	//IClient* Tranlient =NULL;
	//Tranlient = g_pSOServer.GetTranClient();
	g_hThread = ::CreateThread(NULL,0,ThreadProcessTranClient,(void *)m_pTransferClient,0,&dwThreadID);
	g_hThread = ::CreateThread(NULL,0,ThreadSaveDataTask,(void *)m_pTransferClient,0,&dwThreadID);
	//´´½¨Ò»¸ö·¢°üÏß³Ì
	g_hThread = ::CreateThread(NULL,0,ThreadSendClientData,(void *)m_pServer,0,&dwThreadID);
	return TRUE;
}

void KSwordOnLineSever::Release()
{
	
#ifndef _STANDALONE
	CCriticalSection::Owner locker(g_csFlow);
#else
	g_mutexFlow.Lock();
#endif
	
	if (m_pServer)
	{
		ExitAllPlayer();
		// Close service for no one can login again
		//m_pServer->CloseService();
		// Regist callback function to null
		//m_pServer->RegisterMsgFilter( reinterpret_cast< void * >(m_pServer),NULL);
		// Shut down all client and save their data.
		///const char *pChar = NULL;
		//MessageBox(0,"GS:·þÎñÆ÷¼´½«¹Ø±Õ,ÌßµôËùÓÐÍæ¼Ò!!","¾¯¸æ:",MB_OK);	
	}

	if (m_pCoreServerShell)
	{
		m_pCoreServerShell->Release();
		m_pCoreServerShell = NULL;
	}

	if (m_pGameStatus)
	{
		delete [] m_pGameStatus;
		m_pGameStatus = NULL;
	}

	if (m_pServer)
	{
		m_pServer->Cleanup();
		m_pServer->Release();
#ifndef _STANDALONE
		m_pServer = NULL;
#else
		delete m_pServer;
		m_pServer = NULL;
#endif
	}

	if (m_pGatewayClient)
	{
		m_pGatewayClient->Shutdown();
		m_pGatewayClient->Cleanup();
		m_pGatewayClient->Release();
		m_pGatewayClient = NULL;
	}

	if (m_pDatabaseClient)
	{
		m_pDatabaseClient->Shutdown();
		m_pDatabaseClient->Cleanup();
		m_pDatabaseClient->Release();
		m_pDatabaseClient = NULL;
	}

	if (m_pTransferClient)
	{
		m_pTransferClient->Shutdown();
		m_pTransferClient->Cleanup();
		m_pTransferClient->Release();
		m_pTransferClient = NULL;
	}

	if (m_pChatClient)
	{
		m_pChatClient->Shutdown();
		m_pChatClient->Cleanup();
		m_pChatClient->Release();
		m_pChatClient = NULL;
	}

	if (m_pTongClient)
	{
		m_pTongClient->Shutdown();
		m_pTongClient->Cleanup();
		m_pTongClient->Release();
		m_pTongClient = NULL;
	}

	IP2CONNECTUNIT::iterator it;
	for (it = m_mapIp2TransferUnit.begin(); it != m_mapIp2TransferUnit.end(); ++it)
	{
		KTransferUnit* pUnit = (*it).second;
		if (pUnit)
		{
			delete pUnit;
			pUnit = NULL;
		}
	}
	m_mapIp2TransferUnit.clear();
#ifdef _STANDALONE
	if (net_buffer)
	   delete net_buffer;

	if  (m_pTransferClient)
	{
		delete m_pTransferClient;	m_pTransferClient = NULL;
	}
	if  (m_pDatabaseClient)
	{	delete m_pDatabaseClient;	m_pDatabaseClient = NULL;}
	if (m_pGatewayClient)
	{	delete m_pGatewayClient;	m_pGatewayClient = NULL;}
#endif

#ifdef _STANDALONE
	g_mutexFlow.Unlock();
#endif
	
}
/////////////////////////////ÎÞÏÞÖ÷Ñ­»·Ïß³Ì////////////////////////////////////////////
BOOL KSwordOnLineSever::Breathe()
{
  if (inGo && m_pCoreServerShell && m_bIsRunning)
  {
//---------------------------½øÈëÑ­»·Ç°,¼ì²âÒ»±éËùÓÐ·þÎñÆ÷µÄÁ´½Ó------------
        MessageLoop();     //Íø¹ØÐ­ÒéÑ­»·
		//m_Timer.GetFPS(&s_nFrameRate);4000000
		if (m_nGameLoop>=1512000)
		{//ÖØÆð ¼ÆÊ±Æ÷
			m_nGameLoop = 0 ;
			m_nGameDay++;
			m_Timer.Passed(1);
			printf("-----ÖØÖÃ¼ÆÊ±Æ÷³É¹¦:%lld -----\n",m_Timer.GetElapse());
		}
		LONGLONG nMes = 1000;
		if (nMes * m_nGameLoop <= m_Timer.GetElapse()*GAME_FPS) //1Ãë18Ö¡  Ê±¼äÒç³ö 
		{	
			if (!GetGo())
			{
				Release();
				return FALSE;
			}

			MainLoop();  //Ö÷Ñ­»·

			//Sleep(0);
///////////////////////////////////////////////////////////////////////////////////GetPlayerNumber
			if (m_nGameLoop >= 1080 && (m_nGameLoop%1080)==0)	//Ã¿18Ö¡Ö´ÐÐÒ»´Î Ò»·ÖÖÓÒ»´Î
			{
				//int nFTp=0;
				//m_Timer.GetFPS(&nFTp);
				printf("-----NetCount= %u,GameLoop= %u,Time=%u%s %u:%u %d----- \n",m_pServer->GetClientCount(),m_nGameLoop,m_nGameDay,"d",m_nGameLoop/1080/60,m_nGameLoop/1080%60,0);
				//TongattackLoop();	             //Ã¿Ãë°ï»áÐûÕ½Ñ­»·
			}
/////////////////////////////////////////////////////////////////////////////////////////
		}
		else
		{
#ifndef __linux			//SwitchToThread();
			Sleep(1);
#else
			printf("-------Ê±¼äÅÐ¶ÏÓÐÎó,ÔÝÍ£ 1 Ãë--------- \n");
			Sleep(1);  //Í£Ö¹Ò»Ãë	usleep
#endif
		}
		return TRUE;
	}
	printf("main thread exit\n");
	return TRUE;
}

void KSwordOnLineSever::GatewayLoop()
{
	const char*		pChar = NULL;
	unsigned __int64	uSize = 0;
	try
	{
      while(m_pGatewayClient)
	  {//´ÓBISP»ñÈ¡ÈËÎïÊý¾ÝÊ§°Ü
		pChar = (const char*)m_pGatewayClient->GetPackFromServer(uSize);  //Á´½ÓGateway
		if (!pChar || 0 == uSize)
		{
//			printf("Core MessageLoop »ñÈ¡ÈËÎïÊý¾ÝÊ§°Ü\n");
			break;
		}
		GatewayMessageProcess(pChar, uSize); //´ÓGateway »ñÈ¡ÈËÎïÊý¾Ýµµ°¸
	  }
	}
	catch (...)
	{
		TRACE("Core GatewayLoop Error\n");
	}
}

void KSwordOnLineSever::DatabaseLoop()
{
	const char*		pChar = NULL;
	unsigned __int64	uSize = 0;
	try
	{

#ifndef _STANDALONE
		CCriticalSection::Owner locker(g_csFlow);
#else
		g_mutexFlow.Lock();
#endif

     while(m_pDatabaseClient)
	 {//ÎÞÏÞÑ­»·»ñÈ¡Êý¾Ý¿âÐÅÏ¢(ÅÅÃû ºÍ ´æµµ) ´ÓGODDES
		pChar = (const char*)m_pDatabaseClient->GetPackFromServer(uSize);	  //Ö¸Ïò5001¶Ë¿Ú
		if (!pChar || 0 == uSize)
			break;
		DatabaseMessageProcess(pChar, uSize);
	 }
	   if(!m_pDatabaseClient)
	   {
		   printf("¾¯¸æ:<<<<<<<Êý¾Ý¿â¶ËÁ´½Ó:NULL:DatabaseLoop!>>>>>> ..!\n");
	   }
	}
	catch (...)
	{
		TRACE("Core DatabaseLoop Error\n");
	}
#ifdef _STANDALONE
	g_mutexFlow.Unlock();
#endif
}

void KSwordOnLineSever::TransferLoop()
{
	const char*		pChar = NULL;
	unsigned __int64	uSize = 0;
	try
	{
	m_pCoreServerShell->SendNetMsgToTransfer(m_pTransferClient);

	while(m_pTransferClient)
	{//´ÓS3ÎÞÏÞÑ­»·»ñÈ¡¿ç·þÐÅÏ¢
		pChar = (const char*)m_pTransferClient->GetPackFromServer(uSize);
		if (!pChar || 0 == uSize)
			break;
		TransferMessageProcess(pChar, uSize);
	}

	}
	catch (...)
	{
		TRACE("Core TransferLoop Error\n");
	}
}

void KSwordOnLineSever::ChatLoop()
{
	const char*		pChar = NULL;
	unsigned __int64	uSize = 0;

    try
	{
	m_pCoreServerShell->SendNetMsgToChat(m_pChatClient); 
	 
	while(m_pChatClient)
	{//´ÓS3ÎÞÏÞÑ­»·»ñÈ¡ÁÄÌìÐÅÏ¢
		pChar = (const char*)m_pChatClient->GetPackFromServer(uSize);
		if (!pChar || 0 == uSize)
			break;
		ChatMessageProcess(pChar, uSize); //·¢Ïò¿Í»§¶Ë
	}

	}
	catch (...)
	{
		TRACE("Core ChatLoop Error\n");
	}
}

void KSwordOnLineSever::TongLoop()
{
	const char*		pChar = NULL;
	unsigned __int64	uSize = 0;
	try
	{
	m_pCoreServerShell->SendNetMsgToTong(m_pTongClient);

	while(m_pTongClient)
	{//´ÓS3ÎÞÏÞÑ­»·»ñÈ¡°ï»áÐÅÏ¢
		pChar = (const char*)m_pTongClient->GetPackFromServer(uSize);
		if (!pChar || 0 == uSize)
			break;
		TongMessageProcess(pChar, uSize);
	}

	}
	catch (...)
	{
		TRACE("Core TongLoop Error\n");
	}
}

void KSwordOnLineSever::PlayerLoop()
{
   	int i;
	const char*		pChar = NULL;
	unsigned __int64	uSize = 0;
	try
	{
      for (i = 0; i < m_nMaxGameStaus; ++i) //m_nMaxPlayer	 2000
	  {//×î´óÍæ¼ÒÏÞÖÆ
		if (enumNetUnconnect == GetNetStatus(i))
			continue;
	
		 int nCurLoop=0;
		 while(m_pServer)
		 {//½ÓÊÜ¿Í»§¶ËÍæ¼ÒÏûÏ¢ ·¢Íù---¸÷¸ö·þÎñÆ÷
			 ++nCurLoop;
			if (nCurLoop>m_nMaxClienLoop)
				break;	//Ìø³öÕâ¸öÑ­»· Ã¿¸öÈË¿ÉÒÔÁ¬Ðø×î´ó½ÓÊÕ10´Î°ü
			pChar = (const char*)m_pServer->GetPackFromClient(i, uSize);
			if (!pChar || 0 == uSize)
				break;

			PlayerFschatProcess(i,pChar,uSize);
            PlayerFsMessageProcess(i,pChar,uSize);
		 }
	  } 
	}
	catch (...)
	{
		TRACE("Core PlayerLoop Error\n");
	}
}



//--------------------------ÎÞÏÞÑ­»·»ñÈ¡-Íø¹ØÐ­ÒéÑ­»·----------------------------------
void KSwordOnLineSever::MessageLoop()
{
	int i;
	const char*	pChar = NULL;
	size_t	uSize = 0;
	int    nTiemLoop=0;
	try
	{

/*#ifndef _STANDALONE
	CCriticalSection::Owner locker(g_csFlow);
#else
	g_mutexFlow.Lock();
#endif*/

	while(m_pGatewayClient)
	{//´ÓBISP»ñÈ¡ÈËÎïÊý¾ÝÊ§°Ü
	  nTiemLoop++;
	  if (m_nMaxClienLoop>0 && nTiemLoop>m_nMaxClienLoop)
	  {
		  //printf("---------Ñ­»·Ì«¾Ã(%d) Ìø³öÑ­»·!----------- \n",m_pGameStatus[i].nNetidx);
		  break;	//Ìø³öÕâ¸öÑ­»· Ã¿¸öÈË¿ÉÒÔÁ¬Ðø×î´ó½ÓÊÕ10´Î°ü
	  }

	  try
	  {
		pChar = (const char*)m_pGatewayClient->GetPackFromServer(uSize);  //Á´½ÓGateway
		if (!pChar|| 0 ==uSize)
		{
//			printf("Core MessageLoop »ñÈ¡ÈËÎïÊý¾ÝÊ§°Ü\n");
			break;
		}
		 GatewayMessageProcess(pChar, uSize); //´ÓGateway »ñÈ¡ÈËÎïÊý¾Ýµµ°¸
	  }
	  catch (...)
	  { //¿ÉÄÜ¶ÏÏßÁË
           m_pGatewayClient->Shutdown();	 //¹Ø±ÕÁ´½Ó È»ºóÖØÁ¬
           m_GatewayState = FALSE;
		   m_mapClientNet[GATWSTATE] = FALSE;
		   break;
	  } 
	}
	nTiemLoop=0;
	while(m_pDatabaseClient)
	{//ÎÞÏÞÑ­»·»ñÈ¡Êý¾Ý¿âÐÅÏ¢(ÅÅÃû ºÍ ´æµµ)
		nTiemLoop++;
		if (m_nMaxClienLoop>0 && nTiemLoop>m_nMaxClienLoop)
		{
			//printf("---------Ñ­»·Ì«¾Ã(%d) Ìø³öÑ­»·!----------- \n",m_pGameStatus[i].nNetidx);
			break;	//Ìø³öÕâ¸öÑ­»· Ã¿¸öÈË¿ÉÒÔÁ¬Ðø×î´ó½ÓÊÕ10´Î°ü
		}

		try
		{
		  pChar = (const char*)m_pDatabaseClient->GetPackFromServer(uSize);
		  if (!pChar || 0 == uSize)
			break;
		  DatabaseMessageProcess(pChar, uSize);
		}
		catch (...)
		{ //¿ÉÄÜ¶ÏÏßÁË
			m_pDatabaseClient->Shutdown();	 //¹Ø±ÕÁ´½Ó È»ºóÖØÁ¬
            m_pDatabaseClient = FALSE;
			m_mapClientNet[DATASTATE] = FALSE;
			break;
		}  
	}

	m_pCoreServerShell->SendNetMsgToTransfer(m_pTransferClient);	//Ô­À´ÓÐµÄ
	nTiemLoop=0;
	while(m_pTransferClient)
	{
		nTiemLoop++;
		if (m_nMaxClienLoop>0 && nTiemLoop>m_nMaxClienLoop)
		{
			//printf("---------Ñ­»·Ì«¾Ã(%d) Ìø³öÑ­»·!----------- \n",m_pGameStatus[i].nNetidx);
			break;	//Ìø³öÕâ¸öÑ­»· Ã¿¸öÈË¿ÉÒÔÁ¬Ðø×î´ó½ÓÊÕ10´Î°ü
		}
		//´ÓS3ÎÞÏÞÑ­»·»ñÈ¡¿ç·þ»òÖÐ×ªÐÅÏ¢
		pChar = (const char*)m_pTransferClient->GetPackFromServer(uSize);
		if (!pChar || 0 == uSize)
			break;
		TransferMessageProcess(pChar, uSize);
	} 

	m_pCoreServerShell->SendNetMsgToChat(m_pChatClient); 	 	//Ô­À´ÓÐµÄ
	nTiemLoop=0; 
	while(m_pChatClient)
	{
		nTiemLoop++;
		if (m_nMaxClienLoop>0 && nTiemLoop>m_nMaxClienLoop)
		{
			//printf("---------Ñ­»·Ì«¾Ã(%d) Ìø³öÑ­»·!----------- \n",m_pGameStatus[i].nNetidx);
			break;	//Ìø³öÕâ¸öÑ­»· Ã¿¸öÈË¿ÉÒÔÁ¬Ðø×î´ó½ÓÊÕ10´Î°ü
		}

		//´ÓS3ÎÞÏÞÑ­»·»ñÈ¡ÁÄÌìÐÅÏ¢
		pChar = (const char*)m_pChatClient->GetPackFromServer(uSize);
		if (!pChar || 0 == uSize)
			break;
		ChatMessageProcess(pChar, uSize); //·¢Ïò¿Í»§¶Ë
	}

	m_pCoreServerShell->SendNetMsgToTong(m_pTongClient);		//Ô­À´ÓÐµÄ
	nTiemLoop=0;
	while(m_pTongClient)
	{//´ÓS3ÎÞÏÞÑ­»·»ñÈ¡°ï»áÐÅÏ¢
		nTiemLoop++;
		if (m_nMaxClienLoop>0 && nTiemLoop>m_nMaxClienLoop)
		{
			//printf("---------Ñ­»·Ì«¾Ã(%d) Ìø³öÑ­»·!----------- \n",m_pGameStatus[i].nNetidx);
			break;	//Ìø³öÕâ¸öÑ­»· Ã¿¸öÈË¿ÉÒÔÁ¬Ðø×î´ó½ÓÊÕ10´Î°ü
		}

		pChar = (const char*)m_pTongClient->GetPackFromServer(uSize);
		if (!pChar || 0 == uSize)
			break;
		TongMessageProcess(pChar, uSize);
	}

	pChar = NULL;
	uSize = 0;
	  for (i = 0; i < m_nMaxGameStaus; ++i) //m_nMaxPlayer MAX_PLAYER m_nMaxGameStaus
	  {//×î´óÍæ¼ÒÏÞÖÆ
		  if (enumNetUnconnect == GetNetStatus(i))  //ÒÑ¾­¶ÏÏßµÄ
		 	continue;

		  if (m_pGameStatus[i].nNetidx==-1)         //ÒÑ¾­¶ÏÏßµÄ
			 continue;
		  
		  if (m_pServer==NULL)
		  {  
			 printf("---------m_pServer=NULL,·þÎñÆ÷GSÒÑ¾­¶Ï¿ª!----------- \n");
			 continue;
		  } 

		 int nCurLoop=0;
		 while(m_pServer)
		 {//½ÓÊÜ¿Í»§¶ËÍæ¼ÒÏûÏ¢ ·¢Íù---¸÷¸ö·þÎñÆ÷  ½ÓÍêÒ»¸öÈË  ÔÙµ½ÏÂÒ»¸öÈË
			
			if (m_nMaxClienLoop>0 && nCurLoop>m_nMaxClienLoop)
			{
				//printf("---------Ñ­»·Ì«¾Ã(%d) Ìø³öÑ­»·!----------- \n",m_pGameStatus[i].nNetidx);
				break;	//Ìø³öÕâ¸öÑ­»· Ã¿¸öÈË¿ÉÒÔÁ¬Ðø×î´ó½ÓÊÕ10´Î°ü
			}

			pChar = (const char*)m_pServer->GetPackFromClient(m_pGameStatus[i].nNetidx, uSize);
			if (!pChar || 0 == uSize)
			{
				break;
			}

			nCurLoop ++;
			PlayerFschatProcess(i, pChar, uSize);
            PlayerFsMessageProcess(i, pChar, uSize);
		 }
	  } 
	}
	catch (...)
	{
		TRACE("Core MessageLoop Error\n");
	}

/*#ifdef _STANDALONE
	g_mutexFlow.Unlock();
#endif*/
	
}
//ÁÄÌìÐÅÏ¢Ïß³Ì
void KSwordOnLineSever::ChatMessageProcess(const char *pChar, size_t nSize)
{
	_ASSERT( pChar && nSize);
#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek(pChar);
#else
	BYTE cProtocol = *(BYTE *)pChar;
#endif	
	int j=0;
	switch (cProtocol)
	{
	case chat_coreitemdata:
		{

			CHAT_GROUPMAN* psR = (CHAT_GROUPMAN*)pChar;

			if  (psR->wChatLength!=202)
				break;
		}
	case chat_coretask:
		{
			//printf("---Ö´ÐÐ½Å±¾ÈÎÎñµ½´ï----\n");
			CHAT_GROUPMAN* psR = (CHAT_GROUPMAN*)pChar;

			if  (psR->wChatLength!=200)
				break;
		}
	break;
	case chat_caoritemtime:
		{
			//printf("---ÎïÆ·Ê±¼ä¼ì²âµ½´ï----\n");
			CHAT_GROUPMAN* psR = (CHAT_GROUPMAN*)pChar;
			if  (psR->wChatLength!=201)
				break;

			/*if (m_pCoreServerShell)
				m_pCoreServerShell->CheckItemTime();*/
		}
		break;
	case chat_relegate:  //´ÓS3 »ñÈ¡µÄÐ­ÒéÍ·  --ÆµµÀÏûÏ¢·¢Íù¿Í»§¶Ë
		{//¸½½ü
			CHAT_RELEGATE* pCR = (CHAT_RELEGATE*)pChar;

			DWORD	theID = 0;
			switch (pCR->TargetCls)
			{
			case tgtcls_team:  // ×é¶Ó
			case tgtcls_fac:   // ÃÅÅÉ
			case tgtcls_msgr:
				theID = pCR->TargetID;
				break;
			case tgtcls_scrn:  //ÊÀ½ç£¿
				 j = FindSame(pCR->TargetID);
				theID = m_pGameStatus[j].nPlayerIndex;
				break;
			default: 
				return;
			}
			//printf("--ÁÄÌìD --\n");
			m_pCoreServerShell->GroupChat(
				m_pChatClient,
				pCR->nFromIP,
				pCR->nFromRelayID,
				pCR->channelid,
				pCR->TargetCls,
				theID,
				pCR + 1,
				pCR->routeDateLength);
		}
		break;
	case chat_groupman: //³ÇÊÐ ºÍ ÊÀ½ç £¬¶ÓÎé ÃÅÅÉ
		//printf("--ÁÄÌìA --\n");
		ChatGroupMan(pChar, nSize);
		break;
	case chat_specman:// Ë½ÁÄ
		{
 //         printf("<<<<<<<½ÓÊÜ´ÓS3·µ»ØµÄË½ÁÄÐÅÏ¢³É¹¦>>>>>> OK..!\n");
		  //printf("--ÁÄÌìB --\n");
		  ChatSpecMan(pChar, nSize);  
		}
		break;
	case chat_everyone:   //GM¹«¸æ
		{
		  //printf("--ÁÄÌìC --\n");
		  CHAT_EVERYONE* pCeo = (CHAT_EVERYONE*)pChar;
		  m_pCoreServerShell->OperationRequest(SSOI_BROADCASTING, (unsigned int)(pCeo + 1), pCeo->wChatLength);
		}
		break;
	default:
		break;
	}
}
//////////////////´ÓS3·þÎñÆ÷»ñÈ¡°ï»áÏà¹ØÐÅÏ¢Ñ­»·Ïß³Ì/////////////////////////////////////////////////
void KSwordOnLineSever::TongMessageProcess(const char *pChar, size_t nSize)
{
	if (!pChar)
		return;

	EXTEND_HEADER* pHeader = (EXTEND_HEADER*)pChar;

	if (pHeader->ProtocolFamily == pf_extend)
	{
		if (pHeader->ProtocolID == extend_s2c_passtosomeone)
		{
			EXTEND_PASSTOSOMEONE* pEps = (EXTEND_PASSTOSOMEONE*)pChar;

			if (CheckPlayerID(pEps->lnID ,pEps->nameid)) //ÍøÂçID
			{
				_ASSERT(sizeof(tagExtendProtoHeader) <= sizeof(EXTEND_PASSTOSOMEONE));
				unsigned long lnID = pEps->lnID;
				size_t pckgsize = sizeof(tagExtendProtoHeader) + pEps->datasize;
				tagExtendProtoHeader* pExHdr = (tagExtendProtoHeader*)(pEps + 1) - 1;
				pExHdr->ProtocolType = s2c_extendfriend;
				pExHdr->wLength = pckgsize - 1;

				m_pServer->PackDataToClient(lnID, pExHdr, pckgsize);
			}
		}
		else if (pHeader->ProtocolID == extend_s2c_passtobevy)
		{
			EXTEND_PASSTOBEVY* pEpb = (EXTEND_PASSTOBEVY*)pChar;

			if (pEpb->playercount > 0)
			{
				_ASSERT(sizeof(tagExtendProtoHeader) <= sizeof(EXTEND_PASSTOBEVY));
				void* pExPckg = pEpb + 1;
				WORD playercount = pEpb->playercount;
				tagPlusSrcInfo* pPlayers = (tagPlusSrcInfo*)((BYTE*)pExPckg + pEpb->datasize);
				size_t pckgsize = sizeof(tagExtendProtoHeader) + pEpb->datasize;
				tagExtendProtoHeader* pExHdr = (tagExtendProtoHeader*)pExPckg - 1;
				pExHdr->ProtocolType = s2c_extendfriend;
				pExHdr->wLength = pckgsize - 1;

				for (WORD i = 0; i < playercount; ++i)
				{
					if (CheckPlayerID(pPlayers[i].lnID, pPlayers[i].nameid))
						m_pServer->PackDataToClient(pPlayers[i].lnID, pExHdr, pckgsize);
				}
			}
		}
	}
	else if (pHeader->ProtocolFamily == pf_alltong)
	{
		// Ð­Òé³¤¶È¼ì²â
		if (nSize < sizeof(EXTEND_HEADER))
		{
			printf("---------½ÓÊÕ°ï»áÐûÕ½ÐÅÏ¢Ê§°ÜA!-----------\n");
			return;
		}
		if (pHeader->ProtocolID >= enumS2C_TONG_NUM)
		{
			printf("---------½ÓÊÕ°ï»áÐûÕ½ÐÅÏ¢Ê§°ÜB!-----------\n");
			return;
		}
		if (g_nTongPCSize[pHeader->ProtocolID] < 0)
		{
			if (nSize <= sizeof(EXTEND_HEADER) + 2)
			{
				printf("---------½ÓÊÕ°ï»áÐûÕ½ÐÅÏ¢Ê§°ÜC!-----------\n");
				return;
			}
			WORD	wLength = *((WORD*)((BYTE*)pChar + sizeof(EXTEND_HEADER)));
			if (wLength != nSize)
			{
				printf("---------½ÓÊÕ°ï»áÐûÕ½ÐÅÏ¢Ê§°ÜD!-----------\n");
				return;
			}
		}
		/*else if (g_nTongPCSize[pHeader->ProtocolID] != nSize)
		{//Ð­Òé¼ì²â
			printf("---------½ÓÊÕ°ï»áÐûÕ½ÐÅÏ¢Ê§°ÜE!-----------\n");
			return;
		}*/
		
		switch (pHeader->ProtocolID)
		{
		case enumS3C_TONG_SERVER_TIEM:
			{//µ±Ç°µÄ±ê×¼Ê±¼ä
				STONG_ATTACKSTIME_SENDBACK	*pInfo = (STONG_ATTACKSTIME_SENDBACK*)pChar;

				if (pInfo==NULL)
				{
					printf("============³ÌÐòÊý¾ÝÓÐÎó,ÇëÁªÏµQQ:25557432========== \n");
					//getchar();
					Release();
					Sleep(5000);
					ExitProcess(0);  //ÍË³ö½ø³Ì
					break;
				}

				char ntnnn[64]={0};
				//PassChecksum nTss;
	            //    nTss.SimplyDecryptPassword(ntnnn,RIUHJFHI);
				JieMa_Decipher(RIUHJFHI,ntnnn);
				//¿ªÊ¼¼ì²âÊÇ·ñµ½ÆÚÁËÊ±¼ä 
				if (atoi(pInfo->m_szTime)>=atoi(ntnnn))
				{
				    //printf("============×¢²áÊ±¼äµ½ÆÚ========== \n");
					//printf("============×¢²áÊ±¼äµ½ÆÚ:%s-%s========== \n",pInfo->m_szTime,ntnnn);
				    Release();
				    Sleep(5000);
				    ExitProcess(0);  //ÍË³ö½ø³Ì
				} 
				    //printf("============×¢²áÊ±¼äµ½ÆÚ:%s-%s========== \n",pInfo->m_szTime,ntnnn);
			}
			break;
		case enumS2C_TONG_ALLLIST_INFO:
			{//»ñÈ¡°ï»áÁÐ±íÐÅÏ¢
				STONG_LIST_INFO_SYNC	*pInfo = (STONG_LIST_INFO_SYNC*)pChar;
				if (pInfo==NULL)
					break;
				if ((int)pInfo->m_dwParam <= 0 || (int)pInfo->m_dwParam >= MAX_PLAYER)
					break;
				
				int nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, pInfo->m_dwParam, 0);
				if (nNetID < 0)
					break;
				//printf("---------½ÓÊÕ°ï»áÁÐ±íÐÅÏ¢³É¹¦A!-----------\n");
				TONG_LIST_INFO_SYNC	sInfo;
				sInfo.ProtocolType		= s2c_extendtong;
				sInfo.m_btMsgId			= enumTONG_SYNC_ID_LIST_INFO;
				sInfo.m_btStateNo		= pInfo->m_btStartNo;   //¿ªÊ¼±àºÅ
				sInfo.m_btCurNum		= pInfo->m_btCurNum;    //µ±Ç°µÄÊýÁ¿
				sInfo.m_Count           = pInfo->m_TongCount;   //×Ü°ï»áµÄÊýÁ¿
				//memcpy(sInfo.m_szTitle, pInfo->m_szTitle, sizeof(pInfo->m_szTitle));
				//memcpy(sInfo.m_szTongName, pInfo->m_szTongName, sizeof(pInfo->m_szTongName));
				memset(sInfo.m_sTongList, 0, sizeof(sInfo.m_sTongList));
				memcpy(sInfo.m_sTongList, pInfo->m_sTongList, sizeof(TONG_ONE_MEMBER_INFO) * sInfo.m_btCurNum);
				sInfo.m_wLength = sizeof(sInfo) - 1 - sizeof(TONG_ONE_MEMBER_INFO) * (defTONG_ONE_PAGE_MAX_NUM - sInfo.m_btCurNum);
				//·¢ÍùÓÎÏ·¿Í»§¶ËÍ¬²½Êý¾Ý
				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sInfo, sInfo.m_wLength + 1);
				 //printf("---------½ÓÊÕ°ï»áÁÐ±íÐÅÏ¢³É¹¦B!-----------\n");
			}
			break;
		case enumS2C_TONG_ATTACK_OVER:
			{
				STONG_ATTACK_SENDBACK	*pOver = (STONG_ATTACK_SENDBACK*)pChar; //½ÓÊÕÊý¾Ý
				if (pOver==NULL)
					break;
				if (m_pCoreServerShell)
				{
					    m_pCoreServerShell->SetAttackCamp(pOver->m_szAttackName,pOver->m_dwParam);
				}

				printf("============°ï»á(%s)ÐûÕ½½áÊø==========\n",pOver->m_szAttackName);
			}
			break;
		case  enumS2C_TONG_ATTACK_SENDBACK:
			{
				STONG_ATTACK_SENDBACK	*pSback = (STONG_ATTACK_SENDBACK*)pChar; //½ÓÊÕÊý¾Ý
				if (pSback==NULL)
					break;

				if (pSback->m_dwParam==1)
				{
				    printf("============°ï»áÐûÕ½³É¹¦==========\n");
					if (m_pCoreServerShell)
				      m_pCoreServerShell->OperationRequest(SSOI_RELOAD_WELCOME_MSG,1,(int)&(pSback->m_szAttackName),(int)&(pSback->m_szDAttackName));           //·¢ËÍ»¶Ó­Óï
				}
				else
				{  	
					if (m_pCoreServerShell)
					   m_pCoreServerShell->SetAttackMsg((int)&(pSback->m_szAttackName),(int)&(pSback->m_szDAttackName),2);
					printf("============°ï»áÐûÕ½Ê§°Ü:ÎÞ´Ë°ï»á»òÕýÔÚÐûÕ½==========\n");

				}
			}
			break;
		case enumS3C_TONG_SYNCITY_INFO:
			{//Í¬²½Æß³ÇÐÅÏ¢
				STONG_CITY_INFO_SYNC *pSync = (STONG_CITY_INFO_SYNC*)pChar; //½ÓÊÕÊý¾Ý
				if (pSync==NULL)
					break;

				for (int i = 0; i < m_nMaxGameStaus; ++i) //m_nMaxPlayer
				{//±éÀú×î´óÍæ¼ÒÏÞÖÆ
					if (enumNetUnconnect == GetNetStatus(i))
						continue;

					int	nPlayerIdx = m_pGameStatus[i].nPlayerIndex;

					if (nPlayerIdx <= 0 || nPlayerIdx >= MAX_PLAYER)
						continue;

					int nNetID;
						nNetID = m_pGameStatus[i].nNetidx;//m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID,nPlayerIdx, 0);//m_pGameStatus[i].nNetidx;//

					if 	(nNetID<0)
						continue;

					m_pCoreServerShell->SetAttackWarCityInfo(pChar,nPlayerIdx); //ÉèÖÃ·þÎñÆ÷¶ËÐÅÏ¢
					//Player[nPlayerIdx].m_cTong.SetAttAckCityInfo(&pSync);    

					//----------ÉèÖÃ¿Í»§¶ËÐÅÏ¢-----------------------------
				    CTONG_CITY_INFO_SYNC	sAinfo;
				    sAinfo.ProtocolType = s2c_extendtong;
				    sAinfo.m_btMsgId = enumTONG_SYNC_ID_CITY_INFO;

					sAinfo.m_WarCityCount =pSync->m_WarCityCount;

					if (sAinfo.m_WarCityCount>7 || sAinfo.m_WarCityCount<=0 )
						sAinfo.m_WarCityCount=7;

				    for(int i=0;i<sAinfo.m_WarCityCount;++i)
					{ //Æß³ÇÐÅÏ¢
					    sAinfo.snWarInfo[i].m_idx=pSync->snWarInfo[i].m_idx;
					    sAinfo.snWarInfo[i].m_mapidx=pSync->snWarInfo[i].m_mapidx;
					    sAinfo.snWarInfo[i].m_levle=pSync->snWarInfo[i].m_levle;
					    sAinfo.snWarInfo[i].m_shushou=pSync->snWarInfo[i].m_shushou;
					    sprintf(sAinfo.snWarInfo[i].m_Mastername,pSync->snWarInfo[i].m_Mastername);
					    sprintf(sAinfo.snWarInfo[i].m_szMapName,pSync->snWarInfo[i].m_szMapName);
					    sprintf(sAinfo.snWarInfo[i].m_Tongmaster,pSync->snWarInfo[i].m_Tongmaster);
					}   
					sAinfo.m_wLength = sizeof(sAinfo) - 1;
				   if (m_pServer) //·¢Íù¿Í»§¶Ë
				     	m_pServer->PackDataToClient(nNetID, &sAinfo, sAinfo.m_wLength + 1);

				}
			}
			break;
		case  enumS2C_TONG_ATTACK_GETBACK:
			{//ÎÞÏÞ½ÓÊÕ Í¬²½°ïÕ½ÐÅÏ¢
			   //STONG_ATTACK_INFO_SYNC	*pSync = (STONG_ATTACK_INFO_SYNC*)pChar; //½ÓÊÕÊý¾Ý
			
				//if (m_pCoreServerShell)
				//	nNetID = m_pCoreServerShell->SetAttackInfo(pChar);

				
				/*
				int	nNetID;
				//	nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID,pSync->m_dwNpcID, 0);
				if 	(nNetID<0)
					break;

				CTONG_ATTTACK_INFO_SYNC	sAinfo;
				sAinfo.ProtocolType = s2c_extendtong;
				sAinfo.m_btMsgId = enumTONG_SYNC_ID_ATTACK_INFO;

				*/
			   /*
				for(int i=0;i<pSync->m_WarCityCount;i++)
				{//Æß³ÇÐÅÏ¢
				  sAinfo.snWarInfo[i].m_idx=pSync->snWarInfo[i].m_idx;
				  sAinfo.snWarInfo[i].m_mapidx=pSync->snWarInfo[i].m_mapidx;
				  sAinfo.snWarInfo[i].m_levle=pSync->snWarInfo[i].m_levle;
				  sAinfo.snWarInfo[i].m_shushou=pSync->snWarInfo[i].m_shushou;
				  sprintf(sAinfo.snWarInfo[i].m_Mastername,pSync->snWarInfo[i].m_Mastername);
				  sprintf(sAinfo.snWarInfo[i].m_szMapName,pSync->snWarInfo[i].m_szMapName);
				  sprintf(sAinfo.snWarInfo[i].m_Tongmaster,pSync->snWarInfo[i].m_Tongmaster);
				} */
				//  sAinfo.m_WarCityCount=pSync->m_WarCityCount;
			/*	  sAinfo.m_AttackState=pSync->m_AttackState;
				  sAinfo.m_AttackTime=pSync->m_AttackTime;
				  sAinfo.m_nTempCamp=pSync->m_nTempCamp;
				  sAinfo.m_nLevel=pSync->m_nLevel;
				  sAinfo.m_YDeathCount=pSync->m_YDeathCount;
	              sAinfo.m_DDeathCount=pSync->m_DDeathCount;//¶ÔÕ½µÄËÀÍö´ÎÊý
				  sAinfo.m_nAttackNum=pSync->m_nAttackNum;	                        // ²ÎÕ½³¡Êý
				  sAinfo.m_nWinNum=pSync->m_nWinNum;								// Ê¤Àû³¡Êý
	              sAinfo.m_nLoseNum=pSync->m_nLoseNum;                              // Êäµô³¡Êý
				  sAinfo.m_nMapidx=pSync->m_CurMapIdx;
				  sprintf(sAinfo.m_szAttackName,pSync->m_szAttackName);
				  sAinfo.m_wLength = sizeof(sAinfo) - 1;
			
				if (m_pServer) //·¢Íù¿Í»§¶Ë
					m_pServer->PackDataToClient(nNetID, &sAinfo, sAinfo.m_wLength + 1);
			  */
				    //m_pCoreServerShell->SetAttackInfo(pSync->m_dwNpcID,pSync->m_AttackState,pSync->m_nTempCamp,pSync->m_szAttackName);
				STONG_ATTACK_INFO_SYNC	*pSync = (STONG_ATTACK_INFO_SYNC*)pChar; //½ÓÊÕ°ïÕ½Êý¾Ý

				if (pSync==NULL)
					break;

				//printf("============½ÓÊÕ°ïÕ½¸üÐÂÐÅÏ¢³É¹¦A ========== \n");
				for (int i = 0; i < m_nMaxGameStaus; ++i) //m_nMaxPlayer
				{//±éÀú×î´óÍæ¼ÒÏÞÖÆ
					if (enumNetUnconnect == GetNetStatus(i))
						continue;

					if (m_pGameStatus[i].nNetidx==-1)
						continue;
					
					int	nPlayerIdx = m_pGameStatus[i].nPlayerIndex;
					
					if (nPlayerIdx <= 0 || nPlayerIdx >= MAX_PLAYER)
						continue;

					
					DWORD nTongID;
					char  nTongName[32];
					int  TongBase;
					     TongBase=m_pCoreServerShell->GetGameData(SGDI_CHECK_TONG,nPlayerIdx,0);

					//printf("============½ÓÊÕ°ïÕ½¸üÐÂÐÅÏ¢³É¹¦B:%d ========== \n",TongBase);

					if  (TongBase==0)
						continue; //ÏÂÒ»¸ö

						 strcpy(nTongName,(const char*)TongBase);  //ÄÚ´æ¸´ÖÆ
						 nTongID=g_FileName2Id(nTongName);

				 if (nTongID == pSync->m_dwParam)
				 {//Èç¹û°ï»áÏàÍ¬
					int nNetID;
					    nNetID = m_pGameStatus[i].nNetidx;//m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID,nPlayerIdx, 0);

						m_pCoreServerShell->SetAttackInfo(pChar,nPlayerIdx);

				    if (nNetID<0)
						continue;
					//Í¬²½µ½¿Í»§¶Ë
				    CTONG_ATTTACK_INFO_SYNC	sAinfo;
				    sAinfo.ProtocolType = s2c_extendtong;
				    sAinfo.m_btMsgId = enumTONG_SYNC_ID_ATTACK_INFO;
				    sAinfo.m_dwParam=pSync->m_dwParam;	 //°ï»áµÄÃû³ÆID
				    sAinfo.m_dwNpcID=pSync->m_dwNpcID;
				    sAinfo.m_AttackState=pSync->m_AttackState;
				    sAinfo.m_AttackTime=pSync->m_AttackTime;
				    sAinfo.m_nTempCamp=pSync->m_nTempCamp;
				    sAinfo.m_nLevel=pSync->m_nLevel;
				    sAinfo.m_YDeathCount=pSync->m_YDeathCount;
	                sAinfo.m_DDeathCount=pSync->m_DDeathCount;//¶ÔÕ½µÄËÀÍö´ÎÊý
				    sAinfo.m_nAttackNum=pSync->m_nAttackNum;	                        // ²ÎÕ½³¡Êý
				    sAinfo.m_nWinNum=pSync->m_nWinNum;								// Ê¤Àû³¡Êý
	                sAinfo.m_nLoseNum=pSync->m_nLoseNum;                              // Êäµô³¡Êý
				    sAinfo.m_nMapidx=pSync->m_CurMapIdx;
				    sprintf(sAinfo.m_szAttackName,pSync->m_szAttackName);
				    sAinfo.m_wLength = sizeof(sAinfo) - 1;
			
				    if (m_pServer) //·¢Íù¿Í»§¶Ë
					    m_pServer->PackDataToClient(nNetID, &sAinfo, sAinfo.m_wLength + 1);
				 } 
				} 
//				printf("-½ÓÊÕ°ï»áÐûÕ½ÐÅÏ¢³É¹¦(NPCdwid:%d,×´Ì¬:%d,ÕóÓª:%d,Ê±¼ä:%d,°ï»á:%s)!-\n",pSync->m_dwNpcID,pSync->m_AttackState,pSync->m_nTempCamp,pSync->m_AttackTime,pSync->m_szAttackName);
			}
			break;
		default:
			break;
		}
	}
	else if (pHeader->ProtocolFamily == pf_tong)
	{
		// Ð­Òé³¤¶È¼ì²â
		if (nSize < sizeof(EXTEND_HEADER))
			return;
		if (pHeader->ProtocolID >= enumS2C_TONG_NUM)
			return;
		if (g_nTongPCSize[pHeader->ProtocolID] < 0)
		{
			if (nSize <= sizeof(EXTEND_HEADER) + 2)
				return;
			WORD	wLength = *((WORD*)((BYTE*)pChar + sizeof(EXTEND_HEADER)));
			if (wLength != nSize)
				return;
		}
		else if (g_nTongPCSize[pHeader->ProtocolID] != nSize)
		{
			return;
		}

		switch (pHeader->ProtocolID)
		{
		case enumS2C_TONG_CREATE_SUCCESS:
			{//´´½¨°ï»á³É¹¦
				STONG_CREATE_SUCCESS_SYNC	*pSync = (STONG_CREATE_SUCCESS_SYNC*)pChar;
				//Í¬²½¸ø¿Í»§¶Ë
				STONG_SERVER_TO_CORE_CREATE_SUCCESS sSuccess;
				sSuccess.m_nCamp = pSync->m_btCamp;
				sSuccess.m_dwPlayerNameID = pSync->m_dwPlayerNameID;
				sSuccess.m_nPlayerIdx = pSync->m_dwParam;
				memcpy(sSuccess.m_szTongName, pSync->m_szTongName, pSync->m_btTongNameLength);
				sSuccess.m_szTongName[pSync->m_btTongNameLength] = 0;
                //Í¨Öª¿Í»§¶ËºÍS3´´½¨ÆµµÀ
				m_pCoreServerShell->OperationRequest(SSOI_TONG_CREATE, (unsigned int)&sSuccess, 0);		
				printf("<<<<<<<(%u)´´½¨°ï»á(%s)³É¹¦>>>>>> OK..!\n",sSuccess.m_dwPlayerNameID, sSuccess.m_szTongName);
			}
			break;
		case enumS2C_TONG_CREATE_FAIL:
			{//´´½¨°ï»áÊ§°Ü
				STONG_CREATE_FAIL_SYNC	*pFail = (STONG_CREATE_FAIL_SYNC*)pChar;
				int		nNetID;
				TONG_CREATE_FAIL_SYNC	sFail;

				sFail.ProtocolType = s2c_extendtong;
				sFail.m_btMsgId = enumTONG_SYNC_ID_CREATE_FAIL;
				sFail.m_btFailId = pFail->m_btFailID;
				sFail.m_wLength = sizeof(sFail) - 1;
				nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, pFail->m_dwParam, 0);
				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sFail, sFail.m_wLength + 1);
			
      			printf("<<<<<<<´´½¨°ï»áÊ§°Ü>>>>>> OK..!\n");
			}
			break;
		case enumS2C_TONG_ADD_MEMBER_SUCCESS:
			{//Ìí¼Ó³ÉÔ±³É¹¦
				STONG_ADD_MEMBER_SUCCESS_SYNC	*pSync = (STONG_ADD_MEMBER_SUCCESS_SYNC*)pChar;
				STONG_SERVER_TO_CORE_ADD_SUCCESS	sAdd;

				sAdd.m_nCamp = pSync->m_btCamp;
				sAdd.m_dwPlayerNameID = pSync->m_dwPlayerNameID;
				sAdd.m_nPlayerIdx = pSync->m_dwParam;
				memcpy(sAdd.m_szMasterName, pSync->m_szMasterName, sizeof(sAdd.m_szMasterName));
				memcpy(sAdd.m_szTitleName, pSync->m_szTitleName, sizeof(sAdd.m_szTitleName));
				memcpy(sAdd.m_szTongName, pSync->m_szTongName, sizeof(sAdd.m_szTongName));
				m_pCoreServerShell->OperationRequest(SSOI_TONG_ADD, (unsigned int)&sAdd, 0);
			}
			break;
		case enumS2C_TONG_ADD_MEMBER_FAIL:
			{//Ìí¼Ó³ÉÔ±Ê§°Ü
			    printf("¾¯¸æ:<<<<<<<°ï»á³ÉÔ±Ìí¼ÓÊ§°Ü!>>>>>> ..!\n");
			 //	STONG_ADD_MEMBER_FAIL_SYNC	*pSync = (STONG_ADD_MEMBER_FAIL_SYNC*)pChar;
			}
			break;

		case enumS2C_TONG_HEAD_INFO: 
			{//½ÓÊÜS3·¢¹ýÀ´µÄ°ï»áÊý¾Ý°ü=·þÎñÆ÷¶Ë·¢ËÍ°ï»áÐÅÏ¢¸ø¿Í»§¶Ë
				STONG_HEAD_INFO_SYNC	*pInfo = (STONG_HEAD_INFO_SYNC*)pChar;
				if ((int)pInfo->m_dwParam <= 0 || (int)pInfo->m_dwParam >= MAX_PLAYER)
					break;
                //»ñÈ¡Ä³ÈËµÄÍøÂçÁ¬½ÓºÅ
				int nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, pInfo->m_dwParam, 0);
				if (nNetID < 0)
					break;

				TONG_HEAD_INFO_SYNC	sInfo;
				sInfo.ProtocolType		= s2c_extendtong;
				sInfo.m_btMsgId			= enumTONG_SYNC_ID_HEAD_INFO;
				sInfo.m_dwNpcID			= pInfo->m_dwNpcID;
				sInfo.m_dwMoney			= pInfo->m_dwMoney;
				sInfo.m_nCredit			= pInfo->m_nCredit;
				sInfo.m_btCamp			= pInfo->m_btCamp;
				sInfo.m_btLevel			= pInfo->m_btLevel;
				sInfo.m_btDirectorNum	= pInfo->m_btDirectorNum;
				sInfo.m_btManagerNum	= pInfo->m_btManagerNum;
				sInfo.m_dwMemberNum		= pInfo->m_dwMemberNum;

				KTabFile nApplyList;
				    char nApplyListPath[128];
					sprintf(nApplyListPath,"\\TongSet\\%s.txt",pInfo->m_szTongName);
				    if (nApplyList.Load(nApplyListPath))
                      sInfo.m_btApplyNum = nApplyList.GetHeight();
					else
                      sInfo.m_btApplyNum = 0;
                nApplyList.Clear();

				KIniFile nTongZhaoMu; 
				    if  (nTongZhaoMu.Load("\\TongSet\\TongSet.ini"))
						sInfo.m_btZhaoMuNum = nTongZhaoMu.GetSectionCount();
					else
                        sInfo.m_btZhaoMuNum = 0;
				nTongZhaoMu.Clear();

				memcpy(sInfo.m_szTongName, pInfo->m_szTongName, sizeof(pInfo->m_szTongName));//°ï»áÃû
				memset(sInfo.m_sMember, 0, sizeof(sInfo.m_sMember));                         //Çå¿Õ
				memcpy(sInfo.m_sMember, pInfo->m_sMember, sizeof(TONG_ONE_LEADER_INFO) * (1 + sInfo.m_btDirectorNum));  //°ïÖ÷ºÍ³¤ÀÏÒ»Æð·¢ËÍ
				sInfo.m_wLength = sizeof(sInfo) - 1 - sizeof(sInfo.m_sMember) + sizeof(TONG_ONE_LEADER_INFO) * (1 + sInfo.m_btDirectorNum);
				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sInfo, sInfo.m_wLength + 1);
			}
			break;

		case enumS2C_TONG_MANAGER_INFO:
			{//»ñÈ¡¶Ó³¤µÄÐÅÏ¢ ·¢Íù¿Í»§¶Ë
				STONG_MANAGER_INFO_SYNC	*pInfo = (STONG_MANAGER_INFO_SYNC*)pChar;
				if ((int)pInfo->m_dwParam <= 0 || (int)pInfo->m_dwParam >= MAX_PLAYER)
					break;

				int nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, pInfo->m_dwParam, 0);
				if (nNetID < 0)
					break;

				TONG_MANAGER_INFO_SYNC	sInfo;
				sInfo.ProtocolType		= s2c_extendtong;
				sInfo.m_btMsgId			= enumTONG_SYNC_ID_MANAGER_INFO;
				sInfo.m_dwMoney			= pInfo->m_dwMoney;
				sInfo.m_nCredit			= pInfo->m_nCredit;
				sInfo.m_btCamp			= pInfo->m_btCamp;
				sInfo.m_btLevel			= pInfo->m_btLevel;
				sInfo.m_btDirectorNum	= pInfo->m_btDirectorNum;
				sInfo.m_btManagerNum	= pInfo->m_btManagerNum;
				sInfo.m_dwMemberNum		= pInfo->m_dwMemberNum;
				sInfo.m_btStateNo		= pInfo->m_btStartNo;
				sInfo.m_btCurNum		= pInfo->m_btCurNum;
				memcpy(sInfo.m_szTongName, pInfo->m_szTongName, sizeof(pInfo->m_szTongName));
				memset(sInfo.m_sMember, 0, sizeof(sInfo.m_sMember));
				memcpy(sInfo.m_sMember, pInfo->m_sMember, sizeof(TONG_ONE_LEADER_INFO) * sInfo.m_btCurNum);
				sInfo.m_wLength = sizeof(sInfo) - 1 - sizeof(TONG_ONE_LEADER_INFO) * (defTONG_ONE_PAGE_MAX_NUM - sInfo.m_btCurNum);
				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sInfo, sInfo.m_wLength + 1);
			}
			break;

		case enumS2C_TONG_MEMBER_INFO:
			{//»ñÈ¡°ïÖÚµÄÐÅÏ¢ ·¢Íù¿Í»§¶Ë
				STONG_MEMBER_INFO_SYNC	*pInfo = (STONG_MEMBER_INFO_SYNC*)pChar;
				if ((int)pInfo->m_dwParam <= 0 || (int)pInfo->m_dwParam >= MAX_PLAYER)
					break;

				int nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, pInfo->m_dwParam, 0);
				if (nNetID < 0)
					break;

				TONG_MEMBER_INFO_SYNC	sInfo;
				sInfo.ProtocolType		= s2c_extendtong;
				sInfo.m_btMsgId			= enumTONG_SYNC_ID_MEMBER_INFO;
				sInfo.m_dwMoney			= pInfo->m_dwMoney;
				sInfo.m_nCredit			= pInfo->m_nCredit;
				sInfo.m_btCamp			= pInfo->m_btCamp;
				sInfo.m_btLevel			= pInfo->m_btLevel;
				sInfo.m_btDirectorNum	= pInfo->m_btDirectorNum;
				sInfo.m_btManagerNum	= pInfo->m_btManagerNum;
				sInfo.m_dwMemberNum		= pInfo->m_dwMemberNum;
				sInfo.m_btStateNo		= pInfo->m_btStartNo;   //¿ªÊ¼±àºÅ
				sInfo.m_btCurNum		= pInfo->m_btCurNum;    //µ±Ç°µÄÊýÁ¿
				memcpy(sInfo.m_szTitle, pInfo->m_szTitle, sizeof(pInfo->m_szTitle));
				memcpy(sInfo.m_szTongName, pInfo->m_szTongName, sizeof(pInfo->m_szTongName));
				memset(sInfo.m_sMember, 0, sizeof(sInfo.m_sMember));
				memcpy(sInfo.m_sMember, pInfo->m_sMember, sizeof(TONG_ONE_MEMBER_INFO) * sInfo.m_btCurNum);
				sInfo.m_wLength = sizeof(sInfo) - 1 - sizeof(TONG_ONE_MEMBER_INFO) * (defTONG_ONE_PAGE_MAX_NUM - sInfo.m_btCurNum);
				//
				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sInfo, sInfo.m_wLength + 1);
			}
			break;

		case enumS2C_TONG_BE_INSTATED:
			{//±»ÈÎÃü
				STONG_BE_INSTATED_SYNC	*pSync = (STONG_BE_INSTATED_SYNC*)pChar;
				if (m_pGameStatus[pSync->m_dwParam].nPlayerIndex <= 0)
					break;
				STONG_SERVER_TO_CORE_BE_INSTATED	sInstated;
				sInstated.m_nPlayerIdx = m_pGameStatus[pSync->m_dwParam].nPlayerIndex;
				sInstated.m_btFigure = pSync->m_btFigure;
				sInstated.m_btPos = pSync->m_btPos;
				memcpy(sInstated.m_szName, pSync->m_szName, sizeof(pSync->m_szName));
				memcpy(sInstated.m_szTitle, pSync->m_szTitle, sizeof(pSync->m_szTitle));
				m_pCoreServerShell->GetGameData(SGDI_TONG_BE_INSTATED, (unsigned int)&sInstated, 0);
			}
			break;

		case enumS2C_TONG_INSTATE:
			{//ÈÎÃüÖ°Î»
				STONG_INSTATE_SYNC	*pSync = (STONG_INSTATE_SYNC*)pChar;
				TONG_INSTATE_SYNC	sInstate;

				if ((int)pSync->m_dwParam <= 0 || (int)pSync->m_dwParam >= MAX_PLAYER)
					break;

				int nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, pSync->m_dwParam, 0);
				if (nNetID < 0)
					break;

				sInstate.ProtocolType		= s2c_extendtong;
				sInstate.m_btMsgId			= enumTONG_SYNC_ID_INSTATE;
				sInstate.m_wLength			= sizeof(sInstate) - 1;
				sInstate.m_dwTongNameID		= pSync->m_dwTongNameID;
				sInstate.m_btSuccessFlag	= pSync->m_btSuccessFlag;
				sInstate.m_btOldFigure		= pSync->m_btOldFigure;
				sInstate.m_btOldPos			= pSync->m_btOldPos;
				sInstate.m_btNewFigure		= pSync->m_btNewFigure;
				sInstate.m_btNewPos			= pSync->m_btNewPos;
				memcpy(sInstate.m_szTitle, pSync->m_szTitle, sizeof(pSync->m_szTitle));
				memcpy(sInstate.m_szName, pSync->m_szName, sizeof(pSync->m_szName));

				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sInstate, sInstate.m_wLength + 1);
			}
			break;

		case enumS2C_TONG_KICK:
			{
				STONG_KICK_SYNC	*pSync = (STONG_KICK_SYNC*)pChar;
				TONG_KICK_SYNC	sKick;

				if ((int)pSync->m_dwParam <= 0 || (int)pSync->m_dwParam >= MAX_PLAYER)
					break;

				int nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, pSync->m_dwParam, 0);
				if (nNetID < 0)
					break;

				sKick.ProtocolType		= s2c_extendtong;
				sKick.m_btMsgId			= enumTONG_SYNC_ID_KICK;
				sKick.m_wLength			= sizeof(sKick) - 1;
				sKick.m_dwTongNameID	= pSync->m_dwTongNameID;
				sKick.m_btSuccessFlag	= pSync->m_btSuccessFlag;
				sKick.m_btFigure		= pSync->m_btFigure;
				sKick.m_btPos			= pSync->m_btPos;
				memcpy(sKick.m_szName, pSync->m_szName, sizeof(pSync->m_szName));

				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sKick, sKick.m_wLength + 1);
			}
			break;

		case enumS2C_TONG_BE_KICKED:
			{
				STONG_BE_KICKED_SYNC	*pSync = (STONG_BE_KICKED_SYNC*)pChar;
				if (m_pGameStatus[pSync->m_dwParam].nPlayerIndex <= 0)
					break;
				STONG_SERVER_TO_CORE_BE_KICKED	sKicked;
				sKicked.m_nPlayerIdx = m_pGameStatus[pSync->m_dwParam].nPlayerIndex;
				sKicked.m_btFigure = pSync->m_btFigure;
				sKicked.m_btPos = pSync->m_btPos;
				memcpy(sKicked.m_szName, pSync->m_szName, sizeof(pSync->m_szName));
				m_pCoreServerShell->GetGameData(SGDI_TONG_BE_KICKED, (unsigned int)&sKicked, 0);
			}
			break;

		case enumS2C_TONG_LEAVE:
			{//ÅÑÀë°ï»á
				STONG_LEAVE_SYNC	*pSync = (STONG_LEAVE_SYNC*)pChar;
				STONG_SERVER_TO_CORE_LEAVE	sLeave;
				sLeave.m_nPlayerIdx = pSync->m_dwParam;
				sLeave.m_bSuccessFlag = pSync->m_btSuccessFlag;
				memcpy(sLeave.m_szName, pSync->m_szName, sizeof(pSync->m_szName));
				m_pCoreServerShell->GetGameData(SGDI_TONG_LEAVE, (unsigned int)&sLeave, 0);
			}
			break;

		case enumS2C_TONG_CHECK_CHANGE_MASTER_POWER:
			{
				STONG_CHECK_GET_MASTER_POWER_SYNC	*pSync = (STONG_CHECK_GET_MASTER_POWER_SYNC*)pChar;
				if (m_pGameStatus[pSync->m_dwParam].nPlayerIndex <= 0)
					break;
				STONG_SERVER_TO_CORE_CHECK_GET_MASTER_POWER	sCheck;
				sCheck.m_dwTongNameID	= pSync->m_dwTongNameID;
				sCheck.m_btFigure		= pSync->m_btFigure;
				sCheck.m_btPos			= pSync->m_btPos;
				sCheck.m_nPlayerIdx		= m_pGameStatus[pSync->m_dwParam].nPlayerIndex;
				memcpy(sCheck.m_szName, pSync->m_szName, sizeof(pSync->m_szName));

				int nRet = 0;
				if (m_pCoreServerShell)
					nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_GET_MASTER_POWER, (unsigned int)&sCheck, 0);

				STONG_ACCEPT_MASTER_COMMAND	sAccept;

				sAccept.ProtocolFamily	= pf_tong;
				sAccept.ProtocolID		= enumC2S_TONG_ACCEPT_MASTER;
				sAccept.m_dwParam		= m_pGameStatus[pSync->m_dwParam].nPlayerIndex;
				sAccept.m_dwTongNameID	= pSync->m_dwTongNameID;
				sAccept.m_btFigure		= pSync->m_btFigure;
				sAccept.m_btPos			= pSync->m_btPos;
				if (nRet)
					sAccept.m_btAcceptFalg = 1;
				else
					sAccept.m_btAcceptFalg = 0;
				memcpy(sAccept.m_szName, pSync->m_szName, sizeof(pSync->m_szName));
				
				if (m_pTongClient)
					m_pTongClient->SendPackToServer((const void*)&sAccept, sizeof(sAccept));
			}
			break;

		case enumS2C_TONG_CHANGE_MASTER_FAIL:
			{//´«Î»Ê§°Ü
				STONG_CHANGE_MASTER_FAIL_SYNC	*pFail = (STONG_CHANGE_MASTER_FAIL_SYNC*)pChar;
				int nPlayer = pFail->m_dwParam;
				switch (pFail->m_btFailID)
				{
				case 0:
					break;
				case 1:
					nPlayer = m_pGameStatus[pFail->m_dwParam].nPlayerIndex;	 //Íæ¼ÒÁ´±íË÷ÒýºÅ
					break;
				default:
					break;
				}

				if (nPlayer <= 0 || nPlayer >= MAX_PLAYER)
					break;
				int nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, nPlayer, 0);
				if (nNetID < 0)
					break;

				TONG_CHANGE_MASTER_FAIL_SYNC	sFail;
				sFail.ProtocolType		= s2c_extendtong;
				sFail.m_btMsgId			= enumTONG_SYNC_ID_CHANGE_MASTER_FAIL;
				sFail.m_wLength			= sizeof(sFail) - 1;
				sFail.m_dwTongNameID	= pFail->m_dwTongNameID;
				sFail.m_btFailID		= pFail->m_btFailID;
				memcpy(sFail.m_szName, pFail->m_szName, sizeof(pFail->m_szName));
				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sFail, sFail.m_wLength + 1);
			}
			break;

		case enumS2C_TONG_CHANGE_AS:
			{
				STONG_CHANGE_AS_SYNC	*pAs = (STONG_CHANGE_AS_SYNC*)pChar;
				if (m_pGameStatus[pAs->m_dwParam].nPlayerIndex <= 0)
					break;
				STONG_SERVER_TO_CORE_CHANGE_AS	sChange;
				sChange.m_nPlayerIdx = m_pGameStatus[pAs->m_dwParam].nPlayerIndex;
				sChange.m_dwTongNameID = pAs->m_dwTongNameID;
				sChange.m_btFigure = pAs->m_btFigure;
				sChange.m_btPos = pAs->m_btPos;
				// ×Ô¼ºµÄÐÂÍ·ÏÎ
				memcpy(sChange.m_szTitle, pAs->m_szTitle, sizeof(pAs->m_szTitle));
				// ÐÂ°ïÖ÷µÄÃû×Ö
				memcpy(sChange.m_szName, pAs->m_szName, sizeof(pAs->m_szName));
				m_pCoreServerShell->GetGameData(SGDI_TONG_CHANGE_AS, (unsigned int)&sChange, 0);
			}
			break;

		case enumS2C_TONG_CHANGE_MASTER:
			{
				STONG_CHANGE_MASTER_SYNC	*pMaster = (STONG_CHANGE_MASTER_SYNC*)pChar;
				STONG_SERVER_TO_CORE_CHANGE_MASTER	sChange;
				sChange.m_dwTongNameID = pMaster->m_dwTongNameID;
				memcpy(sChange.m_szName, pMaster->m_szName, sizeof(pMaster->m_szName));
				m_pCoreServerShell->GetGameData(SGDI_TONG_CHANGE_MASTER, (unsigned int)&sChange, 0);
			}
			break;
		case enumS2C_TONG_MONEY_SAVE:
			{
				STONG_MONEY_SYNC	*pMoney = (STONG_MONEY_SYNC*)pChar;
				STONG_SERVER_TO_CORE_MONEY pChange;
				pChange.m_dwTongNameID = pMoney->m_dwTongNameID;
				pChange.m_dwMoney      = pMoney->m_dwMoney;
				pChange.m_nMoney       = pMoney->m_nMoney;
				pChange.nType          = 0;
				pChange.m_nPlayerIdx   = pMoney->m_dwParam;
				if (pChange.m_nPlayerIdx <= 0)
					break;
				m_pCoreServerShell->GetGameData(SGDI_TONG_CHANGE_MONEY, (unsigned int)&pChange, 0);
			}
			break;
		case enumS2C_TONG_MONEY_GET:
			{
				STONG_MONEY_SYNC	*pMoney = (STONG_MONEY_SYNC*)pChar;
				STONG_SERVER_TO_CORE_MONEY pChange;
				pChange.m_dwTongNameID = pMoney->m_dwTongNameID;
				pChange.m_dwMoney      = pMoney->m_dwMoney;
				pChange.m_nMoney       = pMoney->m_nMoney;
				pChange.nType          = 1;
				pChange.m_nPlayerIdx   = pMoney->m_dwParam;
				if (pChange.m_nPlayerIdx <= 0)
					break;
				m_pCoreServerShell->GetGameData(SGDI_TONG_CHANGE_MONEY, (unsigned int)&pChange, 0);
			}
			break;
		case enumS2C_TONG_MONEY_SND:
			{
				STONG_MONEY_SYNC	*pMoney = (STONG_MONEY_SYNC*)pChar;
				STONG_SERVER_TO_CORE_MONEY pChange;
				pChange.m_dwTongNameID = pMoney->m_dwTongNameID;
				pChange.m_dwMoney      = pMoney->m_dwMoney;
				pChange.m_nMoney       = pMoney->m_nMoney;
				pChange.nType          = 2;
				pChange.m_nPlayerIdx = pMoney->m_dwParam;
				if (pChange.m_nPlayerIdx <= 0)
					break;
				m_pCoreServerShell->GetGameData(SGDI_TONG_CHANGE_MONEY, (unsigned int)&pChange, 0);
			}
			break;

		case enumS2C_TONG_LOGIN_DATA:
			{//µÇÂ¼Ê±ºò ÉèÖÃ°ï»áÐÅÏ¢
				STONG_LOGIN_DATA_SYNC	*pLogin = (STONG_LOGIN_DATA_SYNC*)pChar;
				STONG_SERVER_TO_CORE_LOGIN	sLogin;
				sLogin.m_dwParam	= pLogin->m_dwParam;
				sLogin.m_nFlag		= pLogin->m_btFlag;
				sLogin.m_nCamp		= pLogin->m_btCamp;
				sLogin.m_nFigure	= pLogin->m_btFigure;
				sLogin.m_nPos		= pLogin->m_btPos;
				sLogin.m_nMoney		= pLogin->m_nMoney;
				sLogin.m_nDeathCount= pLogin->m_nDeathCount;
				memcpy(sLogin.m_szTongName, pLogin->m_szTongName, sizeof(pLogin->m_szTongName));
				memcpy(sLogin.m_szTitle, pLogin->m_szTitle, sizeof(pLogin->m_szTitle));
				memcpy(sLogin.m_szMaster, pLogin->m_szMaster, sizeof(pLogin->m_szMaster));
				memcpy(sLogin.m_szName, pLogin->m_szName, sizeof(pLogin->m_szName));

				sLogin.m_AttackState=pLogin->m_AttackState;
				sLogin.m_nTempCamp=pLogin->m_nTempCamp;
				sLogin.m_AttackTime=pLogin->m_AttackTime;
				sLogin.m_nLevel=pLogin->m_nLevel;
			///ogin.m_szAttackName=pLogin->m_szAttackName;
				memcpy(sLogin.m_szAttackName, pLogin->m_szAttackName, sizeof(pLogin->m_szAttackName));

				m_pCoreServerShell->GetGameData(SGDI_TONG_LOGIN, (unsigned int)&sLogin, 0);	 //ÉèÖÃ°ï»áÐÅÏ¢
			}
			break;

		default:
			break;
		}
	}
	else if (pHeader->ProtocolFamily == pf_friend)
	{//ºÃÓÑ
	}
}
////³ÇÊÐ ºÍ ÊÀ½ç £¬¶ÓÎé ÃÅÅÉ
void KSwordOnLineSever::ChatGroupMan(const void *pData, size_t dataLength)
{
	_ASSERT(pData && dataLength);

	CHAT_GROUPMAN*	pCgc = (CHAT_GROUPMAN *)pData;
	_ASSERT(pCgc->wSize + 1 == dataLength);

	_ASSERT(sizeof(tagExtendProtoHeader) <= sizeof(CHAT_GROUPMAN));
	void* pExPckg = pCgc + 1;
	BYTE hasIdentify = pCgc->byHasIdentify;
	WORD playercount = pCgc->wPlayerCount;
	void* pPlayersData = (tagPlusSrcInfo*)((BYTE*)pExPckg + pCgc->wChatLength);
	size_t pckgsize = sizeof(tagExtendProtoHeader) + pCgc->wChatLength;
	tagExtendProtoHeader* pExHdr = (tagExtendProtoHeader*)pExPckg - 1;
	pExHdr->ProtocolType = s2c_extendchat;
	pExHdr->wLength = pckgsize - 1;

	if (hasIdentify)
	{
		tagPlusSrcInfo* pPlayers = (tagPlusSrcInfo*)pPlayersData;
		for (int i = 0; i < playercount; ++i)
		{
			//TODO: Òª¼Ó¼ìÑéNameIDºÍlnIDµÄÒ»ÖÂÐÔ
			if (CheckPlayerID(pPlayers[i].lnID, pPlayers[i].nameid))
				m_pServer->PackDataToClient(pPlayers[i].lnID, pExHdr, pckgsize);
		}
	}
	else
	{
		WORD* pPlayers = (WORD*)pPlayersData;
		for (int i = 0; i < playercount; ++i)
		{
			//TODO: ²»ÐèÒª¼Ó¼ìÑéNameIDºÍlnIDµÄÒ»ÖÂÐÔ
			//if (pPlayers[i] >= 0)
				m_pServer->PackDataToClient((unsigned long)pPlayers[i], pExHdr, pckgsize);
		}
	}
}
//´ÓS3»ñÈ¡ÐÅÏ¢·¢Íù¿Í»§¶Ë
void KSwordOnLineSever::ChatSpecMan(const void *pData, size_t dataLength)
{
	_ASSERT(pData && dataLength);

	CHAT_SPECMAN*	pCsm = (CHAT_SPECMAN *)pData;
	_ASSERT(pCsm->wSize + 1 == dataLength);
//¼ì²éÃû×ÖID
	if (!CheckPlayerID(pCsm->lnID, pCsm->nameid))
		return;

	_ASSERT(sizeof(tagExtendProtoHeader) <= sizeof(CHAT_SPECMAN));
	void* pExPckg = pCsm + 1;

	unsigned long lnID = pCsm->lnID;
	size_t pckgsize = sizeof(tagExtendProtoHeader) + pCsm->wChatLength;

	tagExtendProtoHeader* pExHeader = (tagExtendProtoHeader*)(pCsm + 1) - 1;
	pExHeader->ProtocolType = s2c_extendchat;
	pExHeader->wLength      = pckgsize - 1;

	m_pServer->PackDataToClient(lnID, pExHeader, pckgsize);
//	printf("<<<<<<<Ë½ÁÄÐÅÏ¢·¢Íù¿Í»§¶Ë(client)³É¹¦>>>>>> OK..!\n");
}

//½ÓÊÕGODDES·¢»ØµÄ½ÇÉ«Êý¾Ý°ü
void KSwordOnLineSever::DatabaseMessageProcess(const char* pData, size_t dataLength)
{
	_ASSERT(pData&&dataLength);

#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek( pData );
#else
	BYTE cProtocol = *(BYTE *)pData;
#endif	
	try{

	// ´ó°ü´¦Àí£¨ÓÃÓÚÅÅÃûµÄÊý¾Ý£©
	if ( cProtocol < s2c_micfghjkdtropackbegin )
	{
		DatabaseLargePackProcess(pData, dataLength);
		return;
	}

	switch (cProtocol)
	{
    case s2c_rolecheckname_result:
		{
			TCheckNameData *	pnPD = (TCheckNameData *)pData;

			if (pnPD->ulIdentity <= 0 || pnPD->ulIdentity >= MAX_PLAYER)
				break;

            if  (pnPD->nDataLen==1 || pnPD->nDataLen==-1)
			{
				m_pCoreServerShell->SetChangeRoleStatus(pnPD->ulIdentity,pnPD->nDataLen,pnPD->nRoleName);
				printf("[¼ì²â½ÇÉ«Ãû³É¹¦]:ÕËºÅ:%s ½ÇÉ«:%s,Ë÷Òý[%d] ½á¹û:%d Íê³É!\n", pnPD->nAccName, pnPD->nRoleName, pnPD->ulIdentity, pnPD->nDataLen);
			}
		}
		break;
	case s2c_roleserver_saverole_result:
		{
			TProcessData*	pPD = (TProcessData *)pData;

			if  (pPD->nDataLen!=791030)
				break;

			int nIndex = pPD->ulIdentity;

			if (nIndex <= 0 || nIndex >= MAX_PLAYER)
				break;
			printf("[´æµµ³É¹¦]:Íæ¼Ò½ÇÉ«´æµµ Ë÷Òý[%d] Íê³É!\n", nIndex);
			if (m_pCoreServerShell->IsCharacterLiXian(nIndex) != 2)
			{//Õý³£ÍË³öµÄ ²»ÊÇÀëÏßµÄ
			  if (m_pCoreServerShell->GetSaveStatus(nIndex)!=SAVE_DOING)
			  {
			    m_pCoreServerShell->SetSaveStatus(nIndex,SAVE_REQUEST);
			    m_pCoreServerShell->ClientDisconnect(nIndex);      //ÉèÖÃÀë¿ªÓÎÏ·×´Ì¬--¶Ï¿ª¿Í»§¶ËÁ´½Ó
			  }
			  else
			  {//Èç¹ûÊÇ½»Ò×µÄ ÉèÖÃ´æµµ²»µôÏß
                m_pCoreServerShell->SetSaveStatus(nIndex,SAVE_IDLE);
			  }
			}

		}
		break;
	default:
		printf("Protocol:(%d) -- database error \n", cProtocol);
		break;
	}

	}
	catch (...)
	{
		TRACE("Core DatabaseMessageProcess Error\n");
	}
}
//ÅÅÃûÊý¾Ý
void KSwordOnLineSever::DatabaseLargePackProcess(const char* pData, size_t dataLength)
{
	_ASSERT( pData && dataLength );

#ifndef _STANDALONE
	CBuffer *pBuffer = m_theRecv.PackUp( pData, dataLength );
#else
	char *pBuffer = m_theRecv.PackUp( pData, dataLength );
#endif

	if ( pBuffer )
	{
		try{
#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek( pBuffer->GetBuffer() );
#else
	BYTE cProtocol = *(BYTE *)pBuffer;
#endif	
//		BYTE cProtocol = CPackager::Peek( pBuffer->GetBuffer() );

		switch ( cProtocol )
		{
		case s2c_gamestatistic:	//»ñÈ¡ÅÅÃûÊý¾Ý
			{
#ifndef _STANDALONE
				TProcessData*	pRD	= (TProcessData *)pBuffer->GetBuffer();
#else
				TProcessData*	pRD	= (TProcessData *)pBuffer;
#endif
				//_ASSERT( pRD->nDataLen == sizeof(TGAME_STAT_DATA)); //ÅÅÃû×´Ì¬½á¹¹
				if  (pRD->nDataLen!=sizeof(TGAME_STAT_DATA))
					break;

				if (m_pCoreServerShell)
				{//ÉèÖÃÅÅÃû
					m_pCoreServerShell->SetLadder((void *)pRD->pDataBuffer, pRD->nDataLen);
				}
			}
			break;

		default:
			break;			
		}

	}
	catch (...)
	{
		TRACE("Core DatabaseLargePackProcess Error\n");
	}

#ifndef _STANDALONE
		try
		{
			if( pBuffer != NULL )
			{
				pBuffer->Release();
				pBuffer = NULL; 
			}
		}
		catch(...) 
		{
			//TRACE("SAFE_RELEASE error\n");
		}
#endif
	}
}
//½ÓÊÕS3·þÎñÆ÷´«¹ýÀ´µÄÊý¾Ý
///////////////////////////////////////////////////////////////////////////////
void KSwordOnLineSever::TransferMessageProcess(const char* pChar, size_t nSize)
{
//#ifndef _STANDALONE
	_ASSERT(pChar && nSize && nSize < 1024);

	try
	{

	EXTEND_HEADER	*pEH = (EXTEND_HEADER *)pChar;

	if (nSize < sizeof(EXTEND_HEADER))
		return;

	// ÊÇ·ñÑ°Â·°ü£¿
	if (pEH->ProtocolID == relay_c2c_askwaydata && pEH->ProtocolFamily == pf_relay)
	{
		//printf("---ÊÕµ½s3·¢À´µÄ¿ç·þÐÅÏ¢relay_c2c_askwaydata --\n");
		TransferAskWayMessageProcess(pChar, nSize);
		return;
	}

	RELAY_DATA	*pRD = (RELAY_DATA *)pEH;
	if (nSize <= sizeof(RELAY_DATA) || nSize != pRD->routeDateLength + sizeof(RELAY_DATA))
		return;

	// ÊÇ·ñÑ°Â·Ê§°Ü°ü
	if (pEH->ProtocolID == relay_s2c_loseway && pEH->ProtocolFamily == pf_relay)
	{
		//printf("---ÊÕµ½s3·¢À´µÄ¿ç·þÐÅÏ¢relay_s2c_loseway --\n");
		TransferLoseWayMessageProcess(pChar + sizeof(RELAY_DATA), nSize - sizeof(RELAY_DATA));
		return;
	}

	if (pEH->ProtocolID != relay_c2c_data && pEH->ProtocolFamily != pf_relay)
		return;

	KTransferUnit* pUnit = NULL;
	IP2CONNECTUNIT::iterator it = m_mapIp2TransferUnit.find(pRD->nFromRelayID);

	if (it != m_mapIp2TransferUnit.end())
	{//Èç¹û´æÔÚÁË
		pUnit = (*it).second;
		//_ASSERT(pUnit);
	}
	else
	{
		pUnit = new KTransferUnit(pRD->nFromIP, pRD->nFromRelayID);//¿ç·þµÄ ip ÐÅÏ¢
		m_mapIp2TransferUnit.insert(IP2CONNECTUNIT::value_type(pRD->nFromRelayID, pUnit));
	}
	//printf("---ÊÕµ½s3·¢À´µÄ¿ç·þpRD->nFromRelayID:%d  pRD->nFromIP:%d ÐÅÏ¢A --\n",pRD->nFromRelayID,pRD->nFromIP);
#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek(pChar, sizeof(RELAY_DATA));
#else
	BYTE cProtocol = *(BYTE *)(pChar + sizeof(RELAY_DATA));
#endif	

	if ( cProtocol < c2c_micropackbegin )
	{
		//printf("---ÊÕµ½s3·¢À´µÄ¿ç·þÐÅÏ¢A --\n");
		TransferLargePackProcess(pChar + sizeof(RELAY_DATA), pRD->routeDateLength, pUnit);
	}
	else if ( cProtocol > c2c_micropackbegin )
	{
		//printf("---ÊÕµ½s3·¢À´µÄ¿ç·þÐÅÏ¢B --\n");
		TransferSmallPackProcess(pChar + sizeof(RELAY_DATA), pRD->routeDateLength, pUnit);
	}
	else
	{
		//printf("---ÊÕµ½s3·¢À´µÄ¿ç·þÐÅÏ¢C --\n");
		_ASSERT( FALSE && "Error!" );
	}

	}
	catch (...)
	{
		TRACE("Core TransferMessageProcess Error\n");
	}
//#endif
}
//´ó°ü´¦Àí
void KSwordOnLineSever::TransferLargePackProcess(const void *pData, size_t dataLength, KTransferUnit *pUnit)
{
	_ASSERT(pData && dataLength);
	if (!pData || !dataLength)
		return;
	

	try{
#ifndef _STANDALONE
	CBuffer *pBuffer = pUnit->m_RecvData.PackUp( pData, dataLength );
#else
	char *pBuffer = pUnit->m_RecvData.PackUp( pData, dataLength );
#endif
	char	szChar[1024];

	if ( pBuffer )
	{
#ifndef _STANDALONE
		BYTE cProtocol = CPackager::Peek( pBuffer->GetBuffer() );
#else
		BYTE cProtocol = *(BYTE *)pBuffer;
#endif		
		
		switch ( cProtocol )
		{
		case c2c_transferroleinfo:
			{//¸Ä±ä½ÇÉ«ÐÅÏ¢ ×ª·þÐÅÏ¢ //Õâ¸öÓ¦¸ÃÊÇÔÚ Ä¿±ê·þÎñÆ÷Ö´ÐÐµÄÁË
#ifndef _STANDALONE
				tagGuidableInfo *pGI = (tagGuidableInfo *)pBuffer->GetBuffer();
#else
				tagGuidableInfo *pGI = (tagGuidableInfo *)pBuffer;
#endif

				GUID guid;
				memcpy( &guid, &( pGI->guid ), sizeof( GUID ) );
				
				if ( pGI->datalength > 0 )
				{
					const TRoleData *pRoleData = ( const TRoleData * )( pGI->szData );
					/*DWORD	dwCRC = 0;
					dwCRC = CRC32(dwCRC, pRoleData, pRoleData->dwDataLen - 4);
					DWORD	dwCheck = *(DWORD *)(pGI->szData + pRoleData->dwDataLen - 4);
					BOOL	bCRC = (dwCheck == dwCRC);
					*/
					BOOL	bCRC = TRUE;
					int nIdx = 0;
					if (bCRC)
					{//Éè¶¨Íæ¼ÒÊý¾Ý
						nIdx = m_pCoreServerShell->AddCharacter(pGI->nExtPoint, pGI->nChangePoint, (void *)pRoleData, &guid,pGI->nLeftTime,pGI->nVipType,pGI->nGameTime); //¸üÐÂÀ©Õ¹µãµÈ¡£¡£¡£
					}
					else
					{
						FILE *fLog = fopen("crc_transfer_error", "a+b");
						if(fLog) 
						{
							char buffer[255];
							sprintf(buffer, "----\r\n%s\r\b%s\r\n", pRoleData->BaseInfo.szName, pRoleData->BaseInfo.caccname);
							fwrite(buffer, 1, strlen(buffer), fLog);
							fwrite(pRoleData, 1, pRoleData->dwDataLen, fLog);
							fclose(fLog);
						}
						printf("----------×ª·þ:ÈËÎïÊý¾ÝÓÐÎó----------\n");
					}

					RELAY_DATA*	pRD = (RELAY_DATA *)szChar;
					pRD->ProtocolFamily = pf_relay;
					pRD->ProtocolID = relay_c2c_data;
					pRD->nFromIP = 0;
					pRD->nFromRelayID = 0;
					pRD->nToIP      = mTransSerVerNo;//pUnit->GetIp();//ÔÙ·¢ÍùÔ´·þÎñÆ÷
					pRD->nToRelayID = mTransSerVerNo;//pUnit->GetRelayID();
					pRD->routeDateLength = sizeof(tagPermitPlayerExchange);

					tagPermitPlayerExchange *pPPEO = (tagPermitPlayerExchange *)(szChar + sizeof(RELAY_DATA));
					pPPEO->cProtocol = c2c_permitplayerexchangein;
					memcpy(&pPPEO->guid, &guid, sizeof(GUID));
					//Ä¿±ê·þÎñÆ÷ ip ºÍ ¶Ë¿Ú
					pPPEO->dwIp  = TransDwIp;    //m_dwInternetIp;
					pPPEO->wPort = m_nServerPort;//TransPort;//m_nServerPort; //·þÎñÆ÷¶Ë¿Ú
					if (nIdx)
					{
						pPPEO->bPermit = true;
					}
					else
					{
						pPPEO->bPermit = false;
					}
					if (m_pTransferClient)
						m_pTransferClient->SendPackToServer(pRD, sizeof(RELAY_DATA) + sizeof(tagPermitPlayerExchange));
					
					if (nIdx)
					{
						tagRegisterAccount	ra;
						ra.cProtocol = c2s_registeraccount;
						strcpy((char *)ra.szAccountName, pRoleData->BaseInfo.caccname);
						if (m_pGatewayClient)
						    m_pGatewayClient->SendPackToServer(&ra, sizeof(tagRegisterAccount));
						//-----------------------------------------------------------------
						/*tagPermitPlayerLogin ppl;  //·¢Íùbishop µÄÐ­Òé---ÊÇ·ñÍ¬ÒâÍæ¼ÒµÇÂ½
						ppl.cProtocol = c2s_permit_logftihmcginqknd;
						memcpy(&(ppl.guid),&guid,sizeof(GUID));
						strcpy((char *)(ppl.szRoleName),(const char *)(pRoleData->BaseInfo.szName));
						strcpy((char *)(ppl.szAccountName),(const char *)(pRoleData->BaseInfo.caccname));
						ppl.bPermit = true;        //ÊÇ·ñÐí¿ÉÍæ¼ÒµÇÂ½
						m_pCoreServerShell->OperationRequest(SSOI_RELOAD_WELCOME_MSG,0,nIdx);           //·¢ËÍ»¶Ó­Óï		
						//·¢Íùbishop
						if (m_pGatewayClient)
							m_pGatewayClient->SendPackToServer((const void *)&ppl,sizeof(tagPermitPlayerLogin));*/
						//-----------------------------------------------------------------
					}
					printf("----------Í¬Òâ(%d)¿ç·þ·þÎñÆ÷¶Ë¿Ú:%d----------\n",pPPEO->bPermit,m_nServerPort);
				}
				else
				{
//					cout << "Don't find any valid data!" << endl;
		        } 
			}
			break;
		default:
			break;
		}

#ifndef _STANDALONE
		try
		{
			if( pBuffer != NULL )
			{
				pBuffer->Release();
				pBuffer = NULL; 
			}
		}
		catch(...) 
		{
			//TRACE("SAFE_RELEASE error\n");
		}
#endif
    }	

	}
	catch (...)
	{
		TRACE("Core TransferLargePackProcess Error\n");
	}

}
//Ð¡°ü´¦Àí
void KSwordOnLineSever::TransferSmallPackProcess(const void *pData, size_t dataLength, KTransferUnit* pUnit)
{

	_ASSERT(/*pUnit && */pData && dataLength);
	//if (!pUnit || !pData || !dataLength)
	//	return;

	try{

#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek( pData );
#else
	BYTE cProtocol = *(BYTE *)pData;
#endif	
	
	switch ( cProtocol )
	{
	case s2s_broadcast:
		{
			BYTE* pS = (BYTE *)pData;
			m_pCoreServerShell->ProcessBroadcastMessage((const char *)(pS + 1), dataLength - 1);
		}
		break;
	case s2s_execute:
		{//Ö´ÐÐ½Å±¾Ïß³Ì
			BYTE* pS = (BYTE *)pData;
			m_pCoreServerShell->ProcessExecuteMessage((const char *)(pS + 1), dataLength - 1);
		}
		break;
	case c2c_permitplayerexchangein:
		{//Õâ¸öÊÇÔÚÔ´Ä¿±ê»úÆ÷ Ö´ÐÐ
			tagPermitPlayerExchange* pPermit = (tagPermitPlayerExchange *)pData;
			int nIndex, lnID;
			m_pCoreServerShell->GetPlayerIndexByGuid(&pPermit->guid, &nIndex, &lnID);//Í¨¹ýguid »ñÈ¡ÍøÂçºÅ
			if (nIndex == 0 || lnID == -1)
				break;

			int i ;
			    i= FindSame(lnID);
			if (!i)
				break;

			if (m_pGameStatus[i].nExchangeStatus != enumExchangeWaitForGameSvrRespone)
				break;

			tagNotifyPlayerExchange	npe;
			if (pPermit->bPermit)
			{//Ô´·þÎñÆ÷ Í¬Òâ×ª·þ
//				SavePlayerData(nIndex);
				m_pCoreServerShell->RemovePlayerForExchange(nIndex);   //¿ªÊ¼µôÏß´æµµ
				//Í¨ÖªGAMEÁ¬½Ó·þÎñÆ÷ÁË
				npe.cProtocol = s2c_notifyplayerexchange;
				memcpy(&npe.guid, &pPermit->guid, sizeof(GUID));
				npe.nIPAddr   = pPermit->dwIp;
				npe.nPort     = pPermit->wPort;	 //Í¬ÒâµÇÂ¼µÄ¶Ë¿Ú
				m_pServer->SendData(lnID, &npe, sizeof(tagNotifyPlayerExchange));
				//------------------Í¨ÖªÍø¹Ø Õâ¸öÕÊºÅÍË³öÓÎÏ·ÁË
				tagLeaveGame	sLeaveGame;     //Àë¿ªÓÎÏ·Ê±  ¿Û³ýÓÎÏ·µã¿¨
				sLeaveGame.cProtocol = c2s_leavegame;
				sLeaveGame.cCmdType  = HOLDACC_LEAVEGAME;
				char	szName[32],szRoleName[32];
				m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ACCOUNT, (unsigned int)szName, nIndex);
				//sLeaveGame.nExtPoint    = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTPOINTCHANGED, nIndex, 0);  //»ñÈ¡Òª¿Û³ýµÄÀ©Õ¹µã
                //sLeaveGame.nGameExtTime = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTTIMECHANGED, nIndex, 0);
				sLeaveGame.nExtPoint    = m_pCoreServerShell->GetExpData(nIndex,0);
				sLeaveGame.nGameExtTime = m_pCoreServerShell->GetExpData(nIndex,1);
				m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)szRoleName, nIndex);
				//sprintf(sLeaveGame.nClientIp,"127.0.0.1");
				strcpy((char *)sLeaveGame.nClientIp, (char *)szRoleName);
				strcpy((char *)sLeaveGame.szAccountName, (char *)szName);
				if (m_pGatewayClient)
				   m_pGatewayClient->SendPackToServer(&sLeaveGame, sizeof(tagLeaveGame));
				// Í¨Öª S3 Õâ¸öÕÊºÅÍË³öÓÎÏ·ÁË----------------
				tagLeaveGame2 lg2;
				lg2.ProtocolFamily = pf_normal;
				lg2.ProtocolID = c2s_leavegame;
				lg2.cCmdType = HOLDACC_LEAVEGAME;
				strcpy((char *)lg2.szAccountName, (char *)szName);
				lg2.nSelServer = m_SerVerNo;
				if (m_pTransferClient)	 //·¢ÍùS3
					m_pTransferClient->SendPackToServer(&lg2, sizeof(tagLeaveGame2));
				//-------------------------------------------
				m_pGameStatus[i].nExchangeStatus = enumExchangeCleaning;
				m_pGameStatus[i].nPlayerIndex = 0;

				printf("----ÊÕµ½Í¨Öª Í¬Òâ×ª·þÀ²!·¢Íù¿Í»§¶Ë(%d)-----\n",pPermit->wPort);

			}
			else
			{//×ª»»µØÍ¼
				npe.cProtocol = s2c_notifyplayerexchange;
				memset(&npe.guid, 0, sizeof(GUID));
				// if false, ip to 0
				npe.nIPAddr = 0;
				npe.nPort = 0;
				m_pServer->SendData(lnID, &npe, sizeof(tagNotifyPlayerExchange));
				
				//»Ö¸´×ª·þ×´Ì¬
				m_pCoreServerShell->RecoverPlayerExchange(nIndex);

				tagRoleEnterGame reg;
				reg.ProtocolType = c2s_roleserver_lock;
				reg.bLock = true;
				m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)reg.Name, nIndex);
				if (m_pDatabaseClient)
					m_pDatabaseClient->SendPackToServer((const void *)&reg, sizeof(tagRoleEnterGame));

				m_pGameStatus[i].nGameStatus = enumPlayerPlaying;
				m_pGameStatus[i].nExchangeStatus = enumExchangeBegin;
			}
		}
		break;
	case c2c_notifyexchange:
		{//ÊÕµ½Í¨Öª »ñÈ¡Êý¾Ý
			tagSearchWay *pSW = (tagSearchWay *)pData;

			if (pSW->lnID < 0 || pSW->lnID >= m_nMaxGameStaus)
			{
				printf("break 1 (%d)\n",pSW->lnID);
				break;
			}

			if (m_pGameStatus[pSW->lnID].nExchangeStatus != enumExchangeSearchingWay)
			{
				printf("break 2 (%d)\n",pSW->lnID);
				break;
			}
			
			int nIndex = m_pGameStatus[pSW->lnID].nPlayerIndex;
			
			if (nIndex <= 0 || nIndex >= MAX_PLAYER || nIndex != pSW->nIndex)
			{
				printf("break 3\n");
				break;
			}

			if (pSW->dwPlayerID != m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ID, nIndex, 0))
			{
				printf("break 4\n");
				break;
			}

			char	szData[1024]={0};

			RELAY_DATA *pRD = (RELAY_DATA *)szData;
			
			pRD->ProtocolFamily = pf_relay;
			pRD->ProtocolID = relay_c2c_data;
			pRD->nToIP      = mTransSerVerNo;//pUnit->GetIp();//Ä¿±êIP
			pRD->nToRelayID = mTransSerVerNo;//pUnit->GetRelayID();
			pRD->nFromIP = 0;
			pRD->nFromRelayID = 0;           //

			tagGuidableInfo gi;
			//¿ç·þ´æµµ Éú³ÉÐÂµÄÊý¾Ý°ü
			TRoleData* pData = (TRoleData *)m_pCoreServerShell->PreparePlayerForExchange(nIndex);  //»ñÈ¡ÈËÎïµÄÊý¾Ý°ü

			if (!pData)
			{
				printf("break »ñÈ¡ÈËÎïÊý¾ÝÊ§°Ü!\n");
				break;
			}
			//×ªÇøÉèÖÃ
			//pData->BaseInfo.isWaiGua = 2;

			gi.cProtocol    = c2c_transferroleinfo;
			m_pCoreServerShell->GetGuid(nIndex, &gi.guid);
			//´ÓÓÎÏ·ÖÐ»ñÈ¡Ô´Êý¾Ý ÓÃÓÚÔÚ Ä¿±ê·þÎñÆ÷½øÈëÓÎÏ·
			gi.nExtPoint    = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTPOINT, nIndex, 0);  //»ñÈ¡¿ÉÒÔÔùËÍµÄÀ©Õ¹µã
			gi.nChangePoint = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTPOINTCHANGED, nIndex, 0);
            gi.nLeftTime    = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_LEFTTIME, nIndex, 0);
            gi.nVipType     = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_VIPTYPE, nIndex, 0);
			gi.nGameTime    = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTTIMECHANGED, nIndex, 0);

			//pData->dwDataLen += 4;
			gi.datalength   = pData->dwDataLen;
			//DWORD	dwCRC = 0;
			//dwCRC = CRC32(dwCRC, pData, pData->dwDataLen - 4);  //Ð£ÑéÂë
            //Ìæ»»ÄÚ´æÖÐµÄÓÎÏ·Êý¾Ý
			m_theSend.AddData(c2c_transferroleinfo,(const char*)&gi,sizeof(tagGuidableInfo));
			//m_theSend.AddData(c2c_transferroleinfo,(const char*)pData,gi.datalength - 4);
			m_theSend.AddData(c2c_transferroleinfo,(const char*)pData,gi.datalength);
			//m_theSend.AddData(c2c_transferroleinfo,(const char*)&dwCRC,4);
			printf("----ÊÕµ½Í¨Öª ¿ªÊ¼×ª·þ:»ñÈ¡Ô´·þÎñÆ÷Êý¾Ý³É¹¦!-----\n");

#ifndef _STANDALONE
			CBuffer* pBuffer = m_theSend.GetHeadPack(c2c_transferroleinfo,512);
#else
			size_t actLen = 0;
			char* pBuffer = m_theSend.GetHeadPack(c2c_transferroleinfo,actLen,512);
#endif
			while(pBuffer)
			{

#ifndef _STANDALONE
				pRD->routeDateLength = pBuffer->GetUsed();
#else
				pRD->routeDateLength = actLen;
#endif
				_ASSERT(pRD->routeDateLength + sizeof(RELAY_DATA) < 1024);

#ifndef _STANDALONE
				memcpy(szData + sizeof(RELAY_DATA), pBuffer->GetBuffer(), pBuffer->GetUsed());
#else
				memcpy(szData + sizeof(RELAY_DATA), pBuffer, actLen);
#endif
                //·¢ÍùÖÐ×ª·þÎñÆ÷ 
				m_pTransferClient->SendPackToServer(szData, pRD->routeDateLength + sizeof(RELAY_DATA));
#ifndef _STANDALONE
				try
				{
					if (pBuffer)
					{
						pBuffer->Release();
						pBuffer = NULL;
					}
				}
				catch(...) 
				{
					//TRACE("SAFE_RELEASE error\n");
				}
#endif

#ifndef _STANDALONE
				pBuffer = m_theSend.GetNextPack(c2c_transferroleinfo);
#else
				pBuffer = m_theSend.GetNextPack(c2c_transferroleinfo, actLen);
#endif
			}
			m_theSend.DelData(c2c_transferroleinfo);   //É¾³ýÊý¾Ý£¿
#ifndef _STANDALONE
			try
			{
				if (pBuffer)
				{
					pBuffer->Release();
					pBuffer = NULL;
				}
			}
			catch(...) 
			{
				//TRACE("SAFE_RELEASE error\n");
			}
#endif
			m_pGameStatus[pSW->lnID].nExchangeStatus = enumExchangeWaitForGameSvrRespone;
		}
		break;
	}

    }
	catch (...)
	{
		TRACE("Core TransferSmallPackProcess Error\n");
	}
}

void KSwordOnLineSever::PlayerFschatProcess(const unsigned long lnID, const char* pData, size_t dataLength)
{
  	_ASSERT(pData && dataLength);

#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek( pData );
#else
	BYTE cProtocol = *(BYTE *)pData;
#endif	

 try
 { 
	//int nGameStatus = m_pGameStatus[lnID].nGameStatus;  //ÓÎÏ·×´Ì¬
	int nIndex = m_pGameStatus[lnID].nPlayerIndex;      //Íæ¼ÒË÷ÒýÁ´±íºÅ	nPlayeridx
	
	BYTE protocoltype = *(BYTE*)pData;

	if (protocoltype >= _c2s_begin_relay && protocoltype <= _c2s_end_relay)
	{
		IClient* pClient = NULL;
		switch (protocoltype)
		{
		case c2s_extend: pClient = m_pTransferClient;//S3 µÄÖÐ×ª·þÎñÆ÷ 
			break;
		case c2s_extendchat: pClient = m_pChatClient;
			break;
		case c2s_extendfriend: pClient = m_pTongClient;
			break;
		default:
			return;
		};

		if (pClient && nIndex && dataLength > sizeof(tagExtendProtoHeader))
		{		
			size_t exsize = dataLength - sizeof(tagExtendProtoHeader);
			const void* pExPckg = pData + sizeof(tagExtendProtoHeader);

			if (protocoltype == c2s_extendchat)
			{
//				printf(">>>>>>>>>>·¢ËÍÆµµÀÏûÏ¢,Íæ¼ÒÐòºÅ(%d)<<<<<<<<<<...\n", nIndex);
				CHAT_CHANNELCHAT_CMD* pEh = (CHAT_CHANNELCHAT_CMD*)pExPckg;
				if (pEh->ProtocolType == chat_channelchat && pEh->channelid!=0)	//·ÇGMÆµµÀÒª¹ýÂËºÍ¸¶Ç®
				{//¸¶·ÑÁÄÌì
					//printf(">>>>>>>>>>·¢ËÍ(Ë÷Òý)(%d)ÆµµÀ(%u)ÏûÏ¢(%d)<<<<<<<<<<...\n", nIndex,pEh->channelid,pEh->cost);
					if (!m_pCoreServerShell->PayForSpeech(nIndex, pEh->cost))
					{	
						//printf(">>>>>>>>>>·¢ËÍ(ÊÕ·Ñ)ÆµµÀÏûÏ¢Ê§°Ü(%d)<<<<<<<<<<...\n", nIndex);
						return;
					}
//					    printf(">>>>>>>>>>·¢ËÍ(Ãâ·Ñ)ÆµµÀÏûÏ¢,Íæ¼ÒÐòºÅ(%d)<<<<<<<<<<...\n", nIndex);
				}

				CHAT_SOMEONECHAT_CMD* psEh = (CHAT_SOMEONECHAT_CMD*)pExPckg;
				if (psEh->ProtocolType == chat_someonechat)	//·ÇGMÆµµÀÒª¹ýÂËºÍ¸¶Ç®
				{//Ë½ÁÄÁÄÌì
					if (m_pCoreServerShell->GetClientNetConnectIdx(nIndex)<0)
					{//¼ì²â¶Ô·½ÊÇ·ñÔÚÏß,·¢ËÍ·½
						return;
					}
				}
			}

			size_t pckgsize = exsize + sizeof(tagPlusSrcInfo);

			#ifdef WIN32
			void* pPckg2 = _alloca(pckgsize);//ÔÚÕ»ÉÏ·ÖÅäÄÚ´æ£¬ÓÃÍêÂíÉÏÊÍ·Å¡£¡£¡£¡£
			#else
			void* pPckg2 = (new char[pckgsize]); //void
			#endif

			memcpy(pPckg2, pExPckg, exsize);
			tagPlusSrcInfo* pSrcInfo = (tagPlusSrcInfo*)((BYTE*)pPckg2 + exsize);
			pSrcInfo->nameid = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ID, nIndex, 0);//Õâ¸öË÷ÒýµÄÃû×ÖID 
			pSrcInfo->lnID   = m_pGameStatus[lnID].nNetidx;   //lnID; Ã¿¸öGS¶¼ÓÐÒ»¸öÎ¨Ò»µÄÕâ¸öÍøÂçID
			pSrcInfo->nSelSerVer = m_SerVerNo;
	        //return;
			if (pClient)
			   pClient->SendPackToServer(pPckg2, pckgsize);  //¸øS3relay ·¢ÏûÏ¢
            //printf(">>>>>>>>>>·¢ËÍS3relay·þÎñÆ÷ÏûÏ¢(%d)(%d)£¬OK<<<<<<<<<<...\n", pSrcInfo->nameid,*(BYTE*)pData);

		#ifdef WIN32

		#else
			delete (char*)pPckg2;
			(char*)pPckg2=NULL;
        #endif

		}
		return;
	}
}
catch (...)
 { 
		TRACE("Core PlayerFschatProcess Error \n");
 } 

}
//-------------------------»ñÈ¡¿Í»§¶Ë·¢Íù·þÎñÆ÷Íæ¼ÒÏà¹ØÐÅÏ¢--------------------------------------------------
void KSwordOnLineSever::PlayerFsMessageProcess(const unsigned long lnID, const char* pData, size_t dataLength)
{

	_ASSERT(pData && dataLength);

#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek(pData);
#else
	BYTE cProtocol = *(BYTE *)pData;
#endif	
 try
 { 
	int nGameStatus = m_pGameStatus[lnID].nGameStatus;       //ÓÎÏ·×´Ì¬
	int nIndex      = m_pGameStatus[lnID].nPlayerIndex;      //Íæ¼ÒÁ´±íË÷ÒýºÅ
	
	BYTE protocoltype = *(BYTE*)pData;

	if (protocoltype >= _c2s_begin_relay && protocoltype <= _c2s_end_relay)
		return;

	//¼ì²éÐ­ÒéÊÇ·ñºÏ·¨£¨´óÐ¡£©
	if (!m_pCoreServerShell->CheckProtocolSize(pData,dataLength))
	{
		m_pGameStatus[lnID].nErrorLoopCount++;
		if (m_pGameStatus[lnID].nErrorLoopCount>=m_errorCount)
		{//Èç¹û·¢ÉúµÄ´íÎó´óÓÚÄ³¸öÊýÖµ,¾ÍÌßµôÁ¬½Ó
			m_pGameStatus[lnID].nErrorLoopCount=0;
			int nIndex = m_pGameStatus[lnID].nPlayerIndex;
			if (nIndex > 0 && nIndex<MAX_PLAYER)
				m_pCoreServerShell->IsShutdownClient(nIndex);
		}
		return;
	}

	//printf("player msg arrived...%d(%d)\n", nGameStatus,*(BYTE*)pData); //[wxb 2003-7-28]
	switch(nGameStatus)
	{
	case enumPlayerPlaying:  
		if (nIndex)
		{
			if (*(BYTE*)pData == c2s_ping)
			{
				ProcessPingReply(lnID, pData, dataLength);  //½ÓÊÕ¿Í»§¶Ë·¢À´µÄ pingµÄÊ±¼ä
			}
			else if (*(BYTE*)pData == c2s_extendtong)
			{//°ï»áÏà¹Ø

				ProcessPlayerTongMsg(nIndex, pData, dataLength);
//			    printf(">>>>>>>>>>·¢ËÍS3relay·þÎñÆ÷½¨°ïÏûÏ¢(%d)(%d)ÇëÇóÀàÐÍ(%d)£¬OK<<<<<<<<<<...\n", nIndex,*(BYTE*)pData,((STONG_PROTOCOL_HEAD*)pData)->m_btMsgId);
			}
			else 
			{//Íæ¼ÒµÄÆäËûÐ­ÒéÏûÏ¢
				m_pCoreServerShell->ProcessClientMessage(nIndex, pData, dataLength);
				//printf(">>>¿Í»§¶Ë->·þÎñÆ÷>>Íæ¼ÒÐòºÅ(%d)Ð­ÒéÐòºÅ(%d)ÇëÇóÀàÐÍ(%d)£¬OK<<<<<<<<<<...\n", nIndex,*(BYTE*)pData-64,((STONG_PROTOCOL_HEAD*)pData)->m_btMsgId);
			}
		}
		break;
	case enumPlayerExchangingServer://¿ªÊ¼¿ç·þ
		break;
	case enumPlayerSyncEnd:         //Í¬²½Íê±Ï Íæ¼ÒÔö¼Óµ½ÓÎÏ·ÖÐ¡£
		if (ProcessSyncReplyProtocol(lnID, pData, dataLength))
		{//×îºóµÇÂ¼ÓÎÏ·³É¹¦¡£¡£¡£¡£¡£¡£¡£¿ªÊ¼ÓÎÏ·°É
			m_pCoreServerShell->AddPlayerToWorld(nIndex);        //Ôö¼ÓÍæ¼Òµ½ÓÎÏ·ÊÀ½ç
			m_pGameStatus[lnID].nGameStatus = enumPlayerPlaying; //ÓÎÏ·×´Ì¬
			PingClient(lnID);                                    //Í¬²½Íê¾Í¿ªÊ¼·¢PING ¿Í»§¶Ë
			printf("----Íæ¼ÒµÇÂ¼ÓÎÏ·³É¹¦,¿ªÊ¼ÓÎÏ·°É:%d---- \n", nIndex);
			//Í¨Öªbishop ½øÈëÓÎÏ·
			tagEnterGame eg;
			eg.cProtocol = c2s__entergamegkjyiruqine;
			m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ACCOUNT, (unsigned int)eg.szAccountName, nIndex);
			if (m_pGatewayClient)  //·¢ÍùÍø¹Ø·þÎñÆ÷
				m_pGatewayClient->SendPackToServer( ( const void * )&eg, sizeof( tagEnterGame ) );
			//·¢ÍùÖÐ×ª·þÎñÆ÷	S3
			tagEnterGame2 eg2;
			eg2.ProtocolFamily = pf_normal;
			eg2.ProtocolID = c2s__entergamegkjyiruqine;
			strcpy(( char * )eg2.szAccountName, (char *)eg.szAccountName);
			eg2.dwNameID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ID, nIndex, 0);
			m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)eg2.szCharacterName, nIndex);
			eg2.lnID = m_pGameStatus[lnID].nNetidx;//lnID; //ÍøÂçºÅ
			eg2.nSelSerVer = m_SerVerNo;

			if (m_pTransferClient) 
				m_pTransferClient->SendPackToServer( (const void *)&eg2, sizeof(tagEnterGame2) );
			//printf("----Í¨ÖªS3(%s)µÇÂ¼ÓÎÏ·³É¹¦:%d---- \n",eg2.szAccountName,nIndex);
			//Ëø¶¨½ÇÉ«Êý¾Ý¿â
			tagRoleEnterGame reg;
			reg.ProtocolType = c2s_roleserver_lock;
			reg.bLock = true;
			strcpy((char *)reg.Name, (char *)eg2.szCharacterName);
			if (m_pDatabaseClient)
				m_pDatabaseClient->SendPackToServer((const void *)&reg, sizeof(tagRoleEnterGame));

			// µÇÂ¼Ê±»ñÈ¡°ï»áÐÅÏ¢
			int	nPlayerIdx = m_pGameStatus[lnID].nPlayerIndex;

			if (nPlayerIdx > 0 && nPlayerIdx < MAX_PLAYER)
			{
				DWORD	dwTongNameID = m_pCoreServerShell->GetGameData(SGDI_TONG_GET_TONG_NAMEID, 0, nPlayerIdx);
				char	szName[32];
				szName[0] = 0;
				m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)szName, nPlayerIdx);

				if (dwTongNameID > 0 && szName[0])
				{//ÓÐ°ï»áµÄÍæ¼Ò
					STONG_GET_LOGIN_DATA_COMMAND	sLogin;
					sLogin.ProtocolFamily	= pf_tong;
					sLogin.ProtocolID		= enumC2S_TONG_GET_LOGIN_DATA;
					sLogin.m_dwParam		= nPlayerIdx;
					sLogin.m_dwTongNameID	= dwTongNameID;
					strcpy(sLogin.m_szName, szName);
					if (m_pTongClient)
						m_pTongClient->SendPackToServer((const void*)&sLogin, sizeof(sLogin));
					//printf("----Í¨ÖªS3(%s)»ñÈ¡°ï»áÐÅÏ¢³É¹¦:%d---- \n",eg2.szAccountName,nIndex);
				}
				m_pCoreServerShell->GetGameData(SGDI_TONG_SEND_SELF_INFO, 0, nPlayerIdx);
			}
		}
		break;
	case enumPlayerBegin:  //Íæ¼Ò¿ªÊ¼µÇÂ½
		{//»ñÈ¡Íæ¼Ò nUseIdx ÇëÇóÁ´±íºÅ
		    //printf("----Íæ¼Ò¿ªÊ¼µÇÂ¼,ÍøÂçºÅ:%d---- \n", lnID);
			int nIndex = ProcessLoginProtocol(lnID, pData, dataLength);
			//printf("process login %d...\n", nIndex);
			if (nIndex)
			{
				if (SendGameDataToClient(lnID, nIndex))  //·¢°ü¸ø¿Í»§¶Ë
				{
					//printf("process login %d ok...\n", nIndex);
					printf("----Í¨Öª¿Í»§¶ËÍæ¼ÒµÇÂ¼³É¹¦,ÍøÂçË÷ÒýºÅ:%d,Ë÷Òý:%d---- \n", lnID,nIndex);
					m_pGameStatus[lnID].nGameStatus = enumPlayerSyncEnd;
					m_pGameStatus[lnID].nPlayerIndex = nIndex;
				}
				else
				{
					printf("process login %d fail...\n", nIndex);
					m_pGameStatus[lnID].nGameStatus =  enumPlayerBegin;
					m_pGameStatus[lnID].nPlayerIndex = nIndex;
				}
			} 
		}
		break;
	}
} 
 catch (...)
 { 
		TRACE("Core PlayerMessageProcess Error\n");
 } 
}
//´¦Àí¸÷¸ö·þÎñÆ÷ËÍÀ´µÄ´óÐ¡°ü --´Óbishop»ñÈ¡´ó°üÊý¾Ý   Íø¹ØÏß³Ì
void KSwordOnLineSever::GatewayMessageProcess(const char* pData, size_t dataLength)
{
	_ASSERT(pData && dataLength);

#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek( pData );
#else
	BYTE cProtocol = *(unsigned char *)pData;
#endif
	
	if ( cProtocol<s2c_micfghjkdtropackbegin)
	{//´ó°ü´¦Àí
		GatewayLargePackProcess(pData, dataLength);
	}
	else if ( cProtocol>s2c_micfghjkdtropackbegin)
	{//Ð¡°ü´¦Àí
		GatewaySmallPackProcess(pData, dataLength);
	}
	/*else
	{
		_ASSERT( FALSE && "Error!" );
	}*/		
}
//´Óbishop»ñÈ¡´ó°üÊý¾Ý
void KSwordOnLineSever::GatewayLargePackProcess(const void *pData, size_t dataLength)
{
#ifndef _STANDALONE
	_ASSERT(pData && dataLength);
	CBuffer *pBuffer = m_theRecv.PackUp(pData, dataLength);
#else
	char *pBuffer = m_theRecv.PackUp(pData, dataLength);
#endif
	if (pBuffer)
	{
#ifndef _STANDALONE
		BYTE cProtocol = CPackager::Peek(pBuffer->GetBuffer());
#else
		BYTE cProtocol = *(BYTE *)pBuffer;
#endif	
		try{

		switch (cProtocol)
		{
		case s2c_syncogjfkdnvhf_roleinfo_cipher:
			{//ÊÕµ½bishop·¢À´µÄÐ­Òé---
#ifndef _STANDALONE
				tagGuidableInfo *pGI = (tagGuidableInfo *)pBuffer->GetBuffer();
#else
				tagGuidableInfo *pGI = (tagGuidableInfo *)pBuffer;
#endif
				GUID guid;
				memcpy(&guid,&(pGI->guid), sizeof(GUID));
							
				if ( pGI->datalength > 0 )
				{
					const TRoleData *pRoleData = ( const TRoleData * )( pGI->szData );
					/*BOOL	bCRCCheck = TRUE;
					if (pRoleData->dwFriendOffset == pRoleData->dwDataLen - 4 && pRoleData->nFriendCount == 0)
					{//×îºóµÄ ÊÇ ÅóÓÑÁÐ±íµÄÆ«ÒÆ
						// ËµÃ÷ÊÇ¼Ó¹ýÐ£ÑéµÄ£¬ËùÒÔºÃÓÑµÄÆ«ÒÆ±È³¤¶ÈÉÙÁË4¸ö×Ö½Ú
						DWORD	dwCRC = 0;
						dwCRC = CRC32(dwCRC, pRoleData, pRoleData->dwDataLen - 4);
						DWORD	dwCheck = *(DWORD *)(pGI->szData + pRoleData->dwDataLen - 4);
						if (dwCheck != dwCRC)
						{//Ð´½ÇÉ«Êý¾Ý°ü
							FILE *fLog = fopen("crc_db_error", "a+b");
							if(fLog) 
							{
								char buffer[255];
								sprintf(buffer, "----\r\n%s\r\b%s\r\n", pRoleData->BaseInfo.szName, pRoleData->BaseInfo.caccname);
								fwrite(buffer, 1, strlen(buffer), fLog);
								fwrite(pRoleData, 1, pRoleData->dwDataLen, fLog);
								fclose(fLog);
							}
							bCRCCheck = FALSE;
						}
					}	*/				
					//°ÑGSµÄÓÎÏ·¶Ë¿Ú·¢Íù¿Í»§¶Ë 56766
					tagPermitPlayerLogin ppl;  //·¢Íùbishop µÄÐ­Òé---ÊÇ·ñÍ¬ÒâÍæ¼ÒµÇÂ½
					ppl.cProtocol = c2s_permit_logftihmcginqknd;
					memcpy(&(ppl.guid),&guid,sizeof(GUID));
					strcpy((char *)(ppl.szRoleName),(const char *)(pRoleData->BaseInfo.szName));
					strcpy((char *)(ppl.szAccountName),(const char *)(pRoleData->BaseInfo.caccname));
					sprintf(ppl.strRoleNmae,pRoleData->BaseInfo.szName);
					ppl.bPermit = false;
					BOOL nKeepForBt=TRUE;
					for(int nindex = 1;nindex < MAX_PLAYER;++nindex)
					{//¼ì²âËùÓÐÍæ¼Ò ÊÇ·ñÓÐÕËºÅÏàÍ¬µÄÔÚÏß
						if (m_pCoreServerShell->IsCharacterLiXian(nindex) == 2)
						{//Èç¹ûÊÇ·þÎñÆ÷ÀëÏßµÄ
							char szName[32]={0};
							m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)szName, nindex);
							if (strcmpi(pRoleData->BaseInfo.szName,szName)==0) //strcmpi
							{//Èç¹ûÊÇÕâ¸ö½ÇÉ«,¾Í½áÊø½ÇÉ«ÀëÏß 
								printf("---»Ö¸´ÀëÏß×´Ì¬:%d,%s VS %s---\n",nindex,pRoleData->BaseInfo.szName,szName); 
								m_pCoreServerShell->SetCharacterLiXian(nindex,0);
								nKeepForBt=FALSE;
								break;
							}
						 }
					 }		
//          		if (Info.nNumPlayer < m_nMaxGameStaus && bCRCCheck )	// È¥µô×¢ÊÍ,CRC´íµÄÓÃ»§¾ÍÎÞ·¨½øÈëÓÎÏ·ÁË
					//{²úÉú nPlayerIdx ±àºÅ
					   //  printf("---Ê£ÓàÓÎÏ·Ê±¼ä:%d---\n",pGI->nGameTime);   
					int nIdx = m_pCoreServerShell->AddCharacter(pGI->nExtPoint, pGI->nChangePoint, (void *)pRoleData, &guid,pGI->nLeftTime,pGI->nVipType,pGI->nGameTime,nKeepForBt); //¸üÐÂÀ©Õ¹µã µÈÐÅÏ¢¡£¡£¡£
						
					if (nIdx>0)
					{//ÍøÂçÁ´½Ó³É¹¦
						ppl.bPermit = true;   //ÊÇ·ñÐí¿ÉÍæ¼ÒµÇÂ½
						m_pCoreServerShell->OperationRequest(SSOI_RELOAD_WELCOME_MSG,0,nIdx);           //·¢ËÍ»¶Ó­Óï
							//printf("------ÏòÕËºÅ(%d)(%s),·¢ËÍÓÎÏ·»¶ËÍÓï³É¹¦!!------\n",nIdx,pRoleData->BaseInfo.caccname);		
					}					
				//	}·¢Íùbishop
					//printf("·¢°ü-·þÎñÆ÷£¬ÊÇ·ñ(%d)Ðí¿ÉÍæ¼Ò(%d                                                                                                                                       )µÇÂ½\n",ppl.bPermit,nIdx);
					if (m_pGatewayClient)
					   m_pGatewayClient->SendPackToServer((const void *)&ppl,sizeof(tagPermitPlayerLogin));
				}
			}
			break;

		default:
			break;			
		}

	}
	catch (...)
	{
		TRACE("Core GatewayLargePackProcess Error\n");
	}

#ifndef _STANDALONE
		try
		{
			if( pBuffer != NULL )
			{
				pBuffer->Release();
				pBuffer = NULL; 
			}
		}
		catch(...) 
		{
			//TRACE("SAFE_RELEASE error\n");
		}
#endif
	}
}
//´Óbishop»ñÈ¡Ð¡°üÐÅÏ¢,²¢¸üÐÂBISHOPÐÅÏ¢
void KSwordOnLineSever::GatewaySmallPackProcess(const void *pData, size_t dataLength)
{
	_ASSERT( pData && dataLength );

#ifndef _STANDALONE
	BYTE cProtocol = CPackager::Peek( pData );
#else 
	BYTE cProtocol = *(BYTE *)pData;
#endif
/*	printf("-----REAL\ndatalen = %u, data = ", dataLength);
	for (int i = 0; i < 32; i++)
		printf("[%02X] ", ((BYTE *)pData)[i]);
	printf("\n");*/
	try
	{
	switch (cProtocol)
	{
	case s2c_querymapinfo:
		{//¸üÐÂµØÍ¼ÈÝÆ÷ GS¿ªÊ¼Ê± ¸üÐÂ
			//char	sMapID[256]={0};
			//int nMapCount = m_pCoreServerShell->GetGameData(SGDI_LOADEDMAP_ID, (unsigned int)&sMapID, sizeof(sMapID));
			//-------------------------------------------------------------------------------------------------
			/*tagUpdateMapID *pUMI=NULL;
			pUMI = (tagUpdateMapID *)new BYTE[nMapCount * sizeof(char) + sizeof(tagUpdateMapID)]; //×î´óÒ²ÊÇ259
			if (pUMI==NULL)
			{
			   printf("------·ÖÅäµØÍ¼ÊýÁ¿ÄÚ´æ(%d)Ê§°Ü------\n",nMapCount * sizeof(char) + sizeof(tagUpdateMapID));
			   printf("------·ÖÅäµØÍ¼ÊýÁ¿ÄÚ´æ(%d)Ê§°Ü------\n",nMapCount * sizeof(char) + sizeof(tagUpdateMapID));
			   printf("------·ÖÅäµØÍ¼ÊýÁ¿ÄÚ´æ(%d)Ê§°Ü------\n",nMapCount * sizeof(char) + sizeof(tagUpdateMapID));
			}
			memset(pUMI, 0, nMapCount * sizeof(char) + sizeof(tagUpdateMapID)); 
			pUMI->cProtocol = c2s_updatemapinfo;
			pUMI->cMapCount = nMapCount;  //µØÍ¼µÄÊýÁ¿
			memcpy(pUMI->szMapID,sMapID,nMapCount * sizeof(char));
			m_pGatewayClient->SendPackToServer((const void *)pUMI,nMapCount * sizeof(char) + sizeof(tagUpdateMapID));
			//printf("----¼ÓÔØµØÍ¼ÊýÁ¿¸üÐÂBISHOPÐÅÏ¢:%d,Len:%d----\n",nMapCount,nMapCount * sizeof(char) + sizeof(tagUpdateMapID));
	        if(pUMI)
			{
				delete pUMI;
				pUMI = NULL;
			}*/
			//----------------------------------------------------------------------------------------------------------
			tagUpdateMapID pUMI;
			pUMI.cProtocol = c2s_updatemapinfo;
			pUMI.cMapCount = m_SerVerNo;//nMapCount; µ±Ç°·þÎñÆ÷±àºÅ
			m_pGatewayClient->SendPackToServer((const void *)&pUMI,sizeof(tagUpdateMapID));

		}
		break;
		
	case s2c_querygameserverinfo:
		{//´´½¨¸üÐÂ BISHOP µÄ Á¬½Ó»·¾³	IP ¶Ë¿ÚµÈ
			tagGameSvrInfo ni;
			
			ni.cProtocol = c2s_updategameserverinfo;
			
			ni.nIPAddr_Internet = m_dwInternetIp;
			ni.nIPAddr_Intraner = m_dwIntranetIp;
			ni.nPort            = m_nServerPort;
			ni.wCapability      = m_nMaxPlayerCount;  //×î´óÈËÊýÏÞÖÆ
			
			m_pGatewayClient->SendPackToServer( ( const void * )&ni, sizeof( tagGameSvrInfo ) );	
		}
		break;
	case s2c_gwsyn_broadcast://½ÓÊÜBISHOP ×ª·¢¹ýÀ´µÄ ÏûÏ¢ Ö÷ÒªÓÃÓÚ ¸÷¸öGSÖ®¼äµÄÐÅÏ¢Í¨Ñ¶
		GwBoardCastProcess((const char*)pData, dataLength);
		break;
	case s2c_gateway_broadcast:
		GatewayBoardCastProcess((const char*)pData, dataLength);
		break;
	}
	}
	catch (...)
	{
		TRACE("Core GatewaySmallPackProcess Error\n");
	}
}

void KSwordOnLineSever::GwBoardCastProcess(const char* pData, size_t dataLength)
{
	if (dataLength != sizeof(tagMsgInGame))
		return;

	try
	{
		tagMsgInGame	*pGatewayBroadCast;
		pGatewayBroadCast = (tagMsgInGame *)pData;
		switch(pGatewayBroadCast->cCmdType)
		{//Ö´ÐÐ´ÓBISHOP·¢ËÍ¹ýÀ´µÄ ²Ù×÷ ÃüÁî
		case AP_WARNING_ALL_PLAYER_QUIT:
			{//ÉèÖÃ±¾ÇøµÄÄ³¸öÈ«¾ÖMIssµÄÖµ 
				m_pCoreServerShell->OperationRequest(SSOI_SETGOLADMISS, (unsigned int)pGatewayBroadCast->nMgsIdx, pGatewayBroadCast->nValIdx);
			}
			break;
		case AP_NOTIFY_GAMESERVER_SAFECLOSE:
			{//Ìßµô ²»ÊÇ±¾ÇøµÄËùÓÐÍæ¼Ò
				//m_pCoreServerShell->OperationRequest(SSOI_KICKOUTPLAYER, (unsigned int)pGatewayBroadCast->nMgsIdx, pGatewayBroadCast->nValIdx);
				ExitAllPlayerIngame();
				printf("---¿ªÊ¼ÌßµôËùÓÐ¿ç·þµÄÍæ¼Ò----\n");
			}
			break;
		case AP_NOTIFY_ALL_PLAYER:
			break;
		default:
			break;
		}

	}
	catch (...)
	{
		TRACE("Core GwBoardCastProcess Error\n");
	}
}

//Ö´ÐÐ´ÓBISHOP·¢ËÍ¹ýÀ´µÄ ÃüÁî
void KSwordOnLineSever::GatewayBoardCastProcess(const char* pData, size_t dataLength)
{
//#ifndef _STANDALONE
	if (dataLength != sizeof(tagGatewayBroadCast))
		return;
	
	try
	{
	tagGatewayBroadCast	*pGatewayBroadCast;
	pGatewayBroadCast = (tagGatewayBroadCast *)pData;
	switch(pGatewayBroadCast->uCmdType)
	{//Ö´ÐÐ´ÓBISHOP·¢ËÍ¹ýÀ´µÄ ²Ù×÷ ÃüÁî
	case AP_WARNING_ALL_PLAYER_QUIT:
		{//Í¨ÖªÒª°²È«ÀëÏß--ÌßµôËùÓÐÍæ¼Ò
			if  (m_pCoreServerShell)
			    m_pCoreServerShell->OperationRequest(SSOI_BROADCASTING, (unsigned int)pGatewayBroadCast->szData, strlen(pGatewayBroadCast->szData));
			// TODO: Save Player and ExitGame when time out
			printf("all_player_quit\n");

			if (pGatewayBroadCast->iSavePai)
			{
				if  (m_pCoreServerShell)
					m_pCoreServerShell->SavePaiShopData();
				printf("Safe save paishopdata to server\n");
			}

			ExitAllPlayer();           //ÌßµôËùÓÐÍæ¼Ò

			if  (m_pCoreServerShell)
				m_pCoreServerShell->ReleaseDbData();
			//SetRunningStatus(FALSE);   //ÉèÖÃGSÍ£Ö¹ÔËÐÐ
			//ExitAllPlayer();
		}
		break;
	case AP_NOTIFY_GAMESERVER_SAFECLOSE:
		{//°²È«¹Ø±Õ·þÎñÆ÷
			if  (m_pCoreServerShell)
			     m_pCoreServerShell->OperationRequest(SSOI_BROADCASTING, (unsigned int)pGatewayBroadCast->szData, strlen(pGatewayBroadCast->szData));
			printf("Safe close server\n");

			if (pGatewayBroadCast->iSavePai)
			{
				if  (m_pCoreServerShell)
					m_pCoreServerShell->SavePaiShopData();

				printf("Safe save paishopdata to server\n");
			}

			ExitAllPlayer();

			if  (m_pCoreServerShell)
				m_pCoreServerShell->ReleaseDbData();
			//SetRunningStatus(FALSE);//ÉèÖÃGSÍ£Ö¹ÔËÐÐ
		}
		break;
	case AP_NOTIFY_ALL_PLAYER:
		{//Í¨ÖªËùÓÐÍæ¼ÒµÄ¹«¸æ
			if  (m_pCoreServerShell)
			     m_pCoreServerShell->OperationRequest(SSOI_BROADCASTING, (unsigned int)pGatewayBroadCast->szData, strlen(pGatewayBroadCast->szData));
		}
		break;
	default:
		break;
	}

	}
	catch (...)
	{
		TRACE("Core GatewayBoardCastProcess Error\n");
	}
//#endif
}

//////////////////////////ÎÞÏÞÖ÷Ñ­»·////////////////////////////////////////
void KSwordOnLineSever::MainLoop()
{

/*#ifndef _STANDALONE
  CCriticalSection::Owner locker(g_csFlow);
#else
	g_mutexFlow.Lock();
#endif*/

 /*try
 { 
	SavePlayerData();                //´æµµ
 } 
 catch (...)
 { 
	 printf("Core SavePlayerData !\n");

 }*/

 /*try
 { 
	 PlayerLogoutGateway();           //Íø¹Ø¶Ï¿ª Ñ­»·¼ì²â ÊÇ·ñÍË³öÓÎÏ·µÄ
 } 
 catch (...)
 { 
	 printf("Core PlayerLogoutGateway !\n");
	 m_pGatewayClient->Shutdown();	 //¹Ø±ÕÁ´½Ó È»ºóÖØÁ¬
	 m_mapClientNet[GATWSTATE] = FALSE;
	 m_GatewayState = FALSE;
 }
 */
 /*try
 {//¿ç·þ Ñ­»·
	 PlayerExchangeServer();          //¿ç·þ×ª»»µØÍ¼Ê± 
 } 
 catch (...)
 { 
	 printf("Core PlayerExchangeServer !\n");
	 
 }*/

 m_pServer->PreparePackSink();    //×¼±¸·¢°ü£¿
 
 try
 { 
	 m_pCoreServerShell->Breathe();   //·þÎñÆ÷¶ËÀïµÄÖ÷Ñ­»·	 
 } 
 catch (...)
 { 
	 printf("Core Breathe !\n");
	 
 }

	//TongattackLoop();	             //°ï»áÐûÕ½Ñ­»·

// ¶¨ÆÚÏòÊý¾Ý¿â²éÑ¯ÅÅÃûÇé¿ö

 try
 { 
    /*#define	MAX_STAT_QUERY_TIME 18*3600*4           // 262144==1<<18 Ã¿ËÄ¸öÐ¡Ê±²éÑ¯Ò»´Î

	if (m_nGameLoop>=MAX_STAT_QUERY_TIME && (m_nGameLoop%MAX_STAT_QUERY_TIME)==0) 
	{//²éÑ¯ÅÅÃû
		if (m_mapClientNet[DATASTATE] && m_pDatabaseClient)
		{
			TProcessData ProcessData;
			ProcessData.nProtoId = c2s_gamestatistic;
			ProcessData.nDataLen = 1;
			ProcessData.ulIdentity = -1;
			ProcessData.pDataBuffer[0] = 0;
			m_pDatabaseClient->SendPackToServer(&ProcessData, sizeof(TProcessData));
		}
	}
	*/
// ¼ì²éÍæ¼ÒÊÇ·ñºÜ³¤Ê±¼äÃ»ÓÐ·¢PINGÖµÁË£¨1min£©ÍøÂçË÷Òý
	//int lnID = TakeRemainder(m_nGameLoop,m_nMaxGameStaus);//m_nGameLoop%m_nMaxGameStaus;  //Àú±é×î´óÍæ¼ÒÊý
	/*int lnID = m_nGameLoop%m_nMaxGameStaus;	  //Ëæ»úµÄ
	int nIndex = m_pGameStatus[lnID].nPlayerIndex; //

	   if (nIndex > 0 && nIndex < MAX_PLAYER  && m_pGameStatus[lnID].nGameStatus == enumPlayerPlaying)
	   {//ÔÚÏßµÄ¾Í¼ì²â
#define	defMAX_PING_TIMEOUT		60*GAME_FPS
#define	defMAX_PING_INTERVAL	5*GAME_FPS
#define defMAX_CONNECTION_TIMEOUT (15 * GAME_FPS) 

		  if (m_pGameStatus[lnID].nReplyPingTime != 0)	//ÉÏ´Î·µ»ØµÄpingÊ±¼ä
		  {//¿Í»§¶ËÓÐÏìÓ¦µÄ  
			 if (m_nGameLoop - m_pGameStatus[lnID].nSendPingTime > defMAX_PING_INTERVAL)
			 { 
				PingClient(lnID);
				//printf("Send ping to clienta.... \n");
			 } 
		  }  
		  else if (m_pGameStatus[lnID].nReplyPingTime==0 && m_pGameStatus[lnID].nPingLoopCount<m_pingCount && (m_nGameLoop - m_pGameStatus[lnID].nSendPingTime > defMAX_PING_TIMEOUT))
		  {//½ÓÊÜ¿Í»§¶Ë·µ»ØµÄÊ±¼ä²î ¿Í»§¶ËÃ»ÏìÓ¦µÄ
			  m_pGameStatus[lnID].nPingLoopCount++;
			  printf("Send ping to clientb.... \n");
			  PingClient(lnID);	//Ë÷Òý
		  }
		  else if (m_nGameLoop - m_pGameStatus[lnID].nSendPingTime > defMAX_PING_TIMEOUT)
		  {//³¬Ê±ÁË
			  printf("no response from client(%d, %d), ShutdownClient it...\n", m_nGameLoop, m_pGameStatus[lnID].nSendPingTime);
			  //m_pCoreServerShell->ClientDisconnect(nIndex);//ÉèÖÃÀë¿ªÓÎÏ·×´Ì¬--¶Ï¿ª¿Í»§¶ËÁ´½Ó
			  //if (m_pGameStatus[lnID].nNetidx>0)
			  //   SetNetStatus(m_pGameStatus[lnID].nNetidx,enumNetUnconnect,false);             //ÉèÖÃ¶Ï¿ªÁ´½Ó
			  m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);                            //¹Ø±Õ¿Í»§¶Ë
		  } 
	   } 
	   else if (m_pGameStatus[lnID].nNetidx >0 && m_pGameStatus[lnID].nNetStatus == enumNetConnected && m_pGameStatus[lnID].nGameStatus == enumPlayerBegin)
	   {
		   // ¼ì²éÒÑ¾­Á¬½Óµ«ÊÇ»¹Ã»ÓÐ·¢µÇÂ¼Ð­ÒéµÄÍæ¼Ò
		   if (m_nGameLoop - m_pGameStatus[lnID].nSendPingTime > defMAX_CONNECTION_TIMEOUT)
		   {
			   printf("Shutdown non-login client[%u] for timeout\n",m_pGameStatus[lnID].nNetidx);
			   m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);
			   if (m_pGameStatus[lnID].nNetidx>0)
			      SetNetStatus(m_pGameStatus[lnID].nNetidx,enumNetUnconnect,false);             //ÉèÖÃ¶Ï¿ªÁ´½Ó
		   }
	   }

	   if (m_nGameLoop & 0x01)  //Ã¿Ö¡¶¼ÏòËùÓÐ¿Í»§¶Ë·¢°ü  
	   { 
		   m_pServer->SendPackToClient(-1);  //ÏòËùÓÐÁ´½Ó·¢°ü
	   } */

	   ++m_nGameLoop;
 } 
 catch (...)
 { 
	 printf("Core ÆäËû !\n");
	 ++m_nGameLoop;	 
 }


/*#ifdef _STANDALONE
	g_mutexFlow.Unlock();
#endif*/

}
//ÊµÊ±¸üÐÂ°ï»áÐûÕ½Ê±¼ä
void KSwordOnLineSever::TongattackLoop()
{ //Ã¿Ãë°ï»áÐûÕ½Ñ­»·
	try
	{
		STONG_ATTACK_COMMAND	sAttack;
		sAttack.ProtocolFamily	= pf_alltong;
		sAttack.ProtocolID		= enumC2S_TONG_ATTACK;
		if (m_pTongClient)
				m_pTongClient->SendPackToServer((const void*)&sAttack, sizeof(sAttack));
	}
	catch (...)
	{
		printf("--------Í¬²½°ï»á³ö´í------- \n");
	}
		
}



//ping ¿Í»§¶Ë
void KSwordOnLineSever::PingClient(const unsigned long lnID)
{
	_ASSERT( lnID > 0 && lnID < m_nMaxGameStaus);
	_ASSERT(m_pGameStatus[lnID].nNetidx >0 && m_pGameStatus[lnID].nPlayerIndex > 0 && m_pGameStatus[lnID].nPlayerIndex < MAX_PLAYER);	
	//printf("PingClient(%d) called\n", lnID);
	PING_COMMAND	pc;
	pc.m_dwTime = m_nGameLoop;
	pc.ProtocolType = s2c_ping;

	if (m_pServer)	  
	  m_pServer->PackDataToClient(m_pGameStatus[lnID].nNetidx, &pc, sizeof(PING_COMMAND));

	m_pGameStatus[lnID].nReplyPingTime = 0;
	m_pGameStatus[lnID].nSendPingTime = m_nGameLoop;  //±¸·ÝÕâ¸öÊ±¼äºÍ¿Í»§¶ËÍ¨»°µÄÊ±¼ä
}

void KSwordOnLineSever::ProcessPingReply(const unsigned long lnID, const char* pData, size_t dataLength)
{
	_ASSERT(lnID < m_nMaxGameStaus);
	_ASSERT(m_pGameStatus[lnID].nPlayerIndex > 0 && m_pGameStatus[lnID].nPlayerIndex < MAX_PLAYER);

	//printf("receive ping from client...\n");
	if (dataLength != sizeof(PING_CLIENTREPLY_COMMAND))
	{
		printf("ping cmd size not correct, may be Non-offical Client... \n");
		m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);
		return;
	}
	
	PING_CLIENTREPLY_COMMAND*	pPC = (PING_CLIENTREPLY_COMMAND *)pData;
	if (pPC->m_dwReplyServerTime != m_pGameStatus[lnID].nSendPingTime)
	{//Ê±¼äÐ£¶Ô£¬Èç¹û²»¶ÔÔò¹Ø±Õ¿Í»§¶Ë
		printf("wrong time in ping cmd content,kill it... \n");
		m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);
		return;
	}

	m_pGameStatus[lnID].nReplyPingTime = m_nGameLoop;
	// reply client ping to client to announce ping interval;
	PING_COMMAND	pc;
	pc.ProtocolType = s2c_replyclientping;
	pc.m_dwTime = pPC->m_dwClientTime;  //·¢ËÍ¿Í·þ¶Ë·µ»ØµÄÊ±¼ä¸ø¿Í»§¶ËÉè¶¨ºÍ·þÎñÆ÷Í¨»°µÄÊ±¼ä
	m_pServer->PackDataToClient(m_pGameStatus[lnID].nNetidx, &pc, sizeof(PING_COMMAND));
}

// ±£³ÖÁ¬½ÓµÄÍæ¼Ò×Ô¶¯´æÅÌ
/*void KSwordOnLineSever::SavePlayerData()
{
	if (!m_pDatabaseClient)
		return;

	int i = 0;
	
	//TProcessData	sProcessData;

	//sProcessData.nProtoId = c2s_roleserver_saveroleinfo;
	//sProcessData.ulIdentity = -1;

	// ±éÀúÍøÂçÁ¬½Ó
	for (i = 0; i < m_nMaxGameStaus;++i)
	{
		int nIndex = m_pGameStatus[i].nPlayerIndex;
	
		if (GetNetStatus(i) == enumNetUnconnect && !m_pCoreServerShell->IsCharacterLiXian(nIndex)) //ÍøÂ·¶Ï¿ª ¾Í²»ÄÜ´æµµÁË 
			continue;

		if (nIndex <= 0 || nIndex >=MAX_PLAYER)
			continue;

		if (m_pGameStatus[i].nNetidx==-1 && !m_pCoreServerShell->IsCharacterLiXian(nIndex))	//ÒÑ¾­¶ÏÍøµÄÁË
		   	continue;	

		//TRoleData* pData = (TRoleData *)m_pCoreServerShell->SavePlayerData(nIndex); 
		if (m_pCoreServerShell->GetSaveStatus(nIndex) == SAVE_REQUEST)
		{//Á¢¼´´æµµµÄ
			SavePlayerData(nIndex, false);
		}
	}
}
*/
BOOL KSwordOnLineSever::SavePlayerData(int nIndex, bool bUnLock)
{
	if (!m_pDatabaseClient)
	{
		cout << "--------------GSÓëÊý¾Ý¿â¶Ï¿ªÁ´½Ó---------------" << endl;
		return FALSE;
	}

	if (nIndex <= 0 || nIndex >= MAX_PLAYER)  //m_nMaxGameStaus
	{
		//cout << "-------------------GSÈËÎïË÷Òý³¬ÏÞ-----------------------" << endl;
		printf("----------------GSÈËÎïË÷Òý(%d)³¬ÏÞ,´æµµÊ§°Ü------------------\n",nIndex);
		return FALSE;
	}

	TProcessData sProcessData;

	sProcessData.nProtoId   = c2s_rolekfjhu_saveroleinfo;	  //·¢Íùgoddes´æµµ
	sProcessData.ulIdentity = nIndex;
	sProcessData.bLeave     = bUnLock;
	sProcessData.uSelServer = m_SerVerNo;
	//printf("-----------´æµµË÷Òý:%d-----------\n", nIndex);
	TRoleData* pData = (TRoleData *)m_pCoreServerShell->SavePlayerDataAtOnce(nIndex);	//¸³ÖµÁË×îºó´æµµÊ±¼ä
    //cout << "-------½ÇÉ«("<< pData->BaseInfo.szName <<")GS¿ªÊ¼´æµµ:ÎïÆ·("<< pData->nItemCount <<")¸ö------Ë÷Òý:" << nIndex << endl;
	 if (pData->nItemCount<=0)
	 {	 		  
       printf("XXXXXXXXXXXXXXXXXXXXXX(%s)XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX\n",pData->BaseInfo.szName);
	   printf("XXXX´æµµXXXX ¾¯¸æ:ÎïÆ·ÊýÁ¿ÎªNULL,ÊýÁ¿Çë¼ì²é!! XXXXXXXXXXXXXXXXXX\n");
	   printf("XXXX´æµµXXXX ¾¯¸æ:ÎïÆ·ÊýÁ¿ÎªNULL,ÊýÁ¿Çë¼ì²é!! XXXXXXXXXXXXXXXXXX\n");
	   printf("XXXX´æµµXXXX ¾¯¸æ:ÎïÆ·ÊýÁ¿ÎªNULL,ÊýÁ¿Çë¼ì²é!! XXXXXXXXXXXXXXXXXX\n");
	   printf("XXXX´æµµXXXX ¾¯¸æ:ÎïÆ·ÊýÁ¿ÎªNULL,ÊýÁ¿Çë¼ì²é!! XXXXXXXXXXXXXXXXXX\n");
	   printf("XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX\n");
	 }

	//printf("-----------´æµµ³¤¶ÈC:%d-----------\n", pData->dwDataLen);
	if (pData && pData->dwDataLen)
	{	//size_t == unsigned int
		sProcessData.nDataLen = pData->dwDataLen + sizeof(TProcessData) - 1;
		
		/*pData->dwDataLen += 4;
		DWORD	dwCRC = 0;
		dwCRC = CRC32(dwCRC, pData, pData->dwDataLen - 4);*/
		// CRC END	ÈÝÆ÷Ôö¼ÓÊý¾Ý
		//printf("-----------´æµµ³¤¶ÈD:%d-----------\n", pData->dwDataLen);
		m_theSend.AddData(c2s_rolekfjhu_saveroleinfo, (const BYTE*)&sProcessData, sizeof(TProcessData) - 1);
		//printf("-----------´æµµ³¤¶ÈK:%d-----------\n", pData->dwDataLen);

		//m_theSend.AddData(c2s_rolekfjhu_saveroleinfo, (const BYTE*)pData, pData->dwDataLen - 4);
		m_theSend.AddData(c2s_rolekfjhu_saveroleinfo, (const BYTE*)pData, pData->dwDataLen);
		//printf("-----------´æµµ³¤¶ÈL:%d-----------\n", pData->dwDataLen);

		//m_theSend.AddData(c2s_rolekfjhu_saveroleinfo, (const BYTE*)&dwCRC, 4);
		//printf("-----------´æµµ³¤¶ÈE:%d-----------\n", pData->dwDataLen);

#ifndef _STANDALONE
		//printf("-----------´æµµ³¤¶ÈF:%d-----------\n", pData->dwDataLen);
		CBuffer* pBuffer = m_theSend.GetHeadPack(c2s_rolekfjhu_saveroleinfo);
		//printf("-----------´æµµ³¤¶ÈH:%d-----------\n", pData->dwDataLen);
#else
		size_t actLen = 0;
		//printf("-----------´æµµ³¤¶ÈI:%d-----------\n", pData->dwDataLen);
		char* pBuffer = m_theSend.GetHeadPack(c2s_rolekfjhu_saveroleinfo, actLen);
		//printf("-----------´æµµ³¤¶ÈJ:%d-----------\n", pData->dwDataLen);
#endif
		
		while(pBuffer)
		{//·¢Íùgoddes
#ifndef _STANDALONE
			//printf("-----------´æµµ³¤¶ÈA:%d-----------\n", pBuffer->GetUsed());
			m_pDatabaseClient->SendPackToServer(pBuffer->GetBuffer(), pBuffer->GetUsed());
#else
			//printf("-----------´æµµ³¤¶ÈB:%d-----------\n",actLen);
			m_pDatabaseClient->SendPackToServer(pBuffer, actLen);
#endif

#ifndef _STANDALONE
			try
			{
				if (pBuffer)
				{
					pBuffer->Release();
					pBuffer = NULL;
				}
			}
			catch(...) 
			{
				//TRACE("SAFE_RELEASE error\n");
			}
#endif

#ifndef _STANDALONE
			pBuffer = m_theSend.GetNextPack(c2s_rolekfjhu_saveroleinfo);   //ÏÂÒ»¸ö°ü
#else
			pBuffer = m_theSend.GetNextPack(c2s_rolekfjhu_saveroleinfo, actLen);
#endif
		}
		m_theSend.DelData(c2s_rolekfjhu_saveroleinfo);
#ifndef _STANDALONE
		try
		{
			if (pBuffer)
			{
				pBuffer->Release();
				pBuffer = NULL;
			}
		}
		catch(...) 
		{

		}
#endif
		//printf("Saving %s(%d)'s data, size:(%d)...\n", pData->BaseInfo.szName, nIndex, sProcessData.nDataLen);
		//m_pCoreServerShell->SetSaveStatus(nIndex, SAVE_DOING);

		return TRUE;
	}

	return FALSE;
}
//¿ç·þ´¦Àí Íæ¼Ò¸Ä±ä·þÎñÆ÷£¨×ª»»µØÍ¼£©
void KSwordOnLineSever::PlayerExchangeServer()
{
	if (!m_pGatewayClient || !m_pCoreServerShell)
		return;
	try
	{
	int i;

	char szChar[1024]={0};
	// ±éÀúÍøÂçÁ¬½Ó
	for (i = 0; i < m_nMaxGameStaus; ++i)
	{
		int nIndex = m_pGameStatus[i].nPlayerIndex;

		if (nIndex <= 0 || nIndex >= MAX_PLAYER)
			continue;

		//if(m_pCoreServerShell->GetClientNetConnectIdx(nIndex)==-1)
		if (m_pGameStatus[i].nNetidx==-1)
		{//Èç¹ûÊÇµôÏßµÄÁË
			//g_pSOServer.SetNetStatus(i, enumNetUnconnect); //Çå¿ÕÊý¾Ý
			continue;
		}
		//¼ì²âÊÇ·ñ ¿ç·þ×´Ì¬
		if (m_pCoreServerShell->IsPlayerExchangingServer(nIndex))
		{   //×ª»»µØÍ¼
			if (!m_pTransferClient)
			{//Èç¹û×ª·þ·þÎñÆ÷¶Ï¿ªµÄ»° ¾ÍÊÇÍ¨Öª¿Í»§¶Ë ×ª»»µØÍ¼
				tagNotifyPlayerExchange	npe;
				npe.cProtocol = s2c_notifyplayerexchange;
				memset(&npe.guid, 0, sizeof(GUID));
				// if false, ip to 0
				
				npe.nIPAddr = 0;  //¿ç·þµÄ ip
				npe.nPort   = 0;
				//·¢ËÍ¿Í»§¶Ë
				m_pServer->SendData(m_pGameStatus[i].nNetidx, &npe, sizeof(tagNotifyPlayerExchange));
				
				m_pCoreServerShell->RecoverPlayerExchange(nIndex);	//È¡Ïû×ª·þ×´Ì¬

				tagRoleEnterGame reg;
				reg.ProtocolType = c2s_roleserver_lock;
				reg.bLock = true;
				m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)reg.Name, nIndex);
				if (m_pDatabaseClient)
					m_pDatabaseClient->SendPackToServer((const void *)&reg, sizeof(tagRoleEnterGame));

				m_pGameStatus[i].nGameStatus = enumPlayerPlaying;
				m_pGameStatus[i].nExchangeStatus = enumExchangeBegin;

				continue;
			}
			//·ñÔò	¿ªÊ¼¿ç·þ
			m_pGameStatus[i].nGameStatus = enumPlayerExchangingServer;
			//m_pGameStatus[i].nExchangeStatus = enumExchangeBegin;  //Ò»Ö±Ñ­»·
			switch (m_pGameStatus[i].nExchangeStatus)
			{
			case enumExchangeBegin:
				{//¿ªÊ¼×ª·þ
					if (!SavePlayerData(nIndex, true)) //´æµµ
						continue;

					RELAY_ASKWAY_DATA	*pRAD = (RELAY_ASKWAY_DATA *)szChar;

					pRAD->ProtocolFamily = pf_relay;
					pRAD->ProtocolID = relay_c2c_askwaydata;

					pRAD->nFromIP      = m_SerVerNo;// 0.0.0.0  ÄÄ¸ö·þÎñÆ÷ ÐèÒª×ª·þ
					pRAD->nFromRelayID = m_SerVerNo;
					pRAD->seekRelayCount = 0;
					// ÀûÓÃµØÍ¼IDÕÒSERVER
					pRAD->seekMethod = rm_map_id;
					pRAD->wMethodDataLength = 4;
					pRAD->routeDateLength = sizeof(tagSearchWay);

					*(DWORD *)(szChar + sizeof(RELAY_ASKWAY_DATA)) = m_pCoreServerShell->GetExchangeMap(nIndex);

					tagSearchWay *pSW = (tagSearchWay *)(szChar + sizeof(RELAY_ASKWAY_DATA) + pRAD->wMethodDataLength);
//					pSW->ProtocolFamily = pf_gameworld;
//					pSW->ProtocolID = c2c_notifyexchange;
					pSW->cProtocol = c2c_notifyexchange;
					pSW->lnID = i;
					pSW->nIndex = m_pGameStatus[i].nPlayerIndex;
					pSW->dwPlayerID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ID, pSW->nIndex, 0);
					// ask another game svr ip
					if  (m_pTransferClient)
					    m_pTransferClient->SendPackToServer(pRAD, sizeof(RELAY_ASKWAY_DATA) + pRAD->wMethodDataLength + pRAD->routeDateLength);
					
					m_pGameStatus[i].nExchangeStatus = enumExchangeSearchingWay;
					printf("----ÊÕµ½Í¨Öª(%d)¿ªÊ¼×ª·þB!-----\n",i);
				}
				break;
			case enumExchangeSearchingWay:
				break;
			case enumExchangeWaitForGameSvrRespone:
				{

				}
				break;
			case enumExchangeCleaning:
				{
					
				}
				break;
			}
		}
	}
	}
	catch (...)
	{
		TRACE("Core PlayerExchangeServer Error\n");
	}
}
//ÍË³öÍø¹ØµÄ
void KSwordOnLineSever::PlayerLogoutGateway()
{
	if (!m_pGatewayClient || !m_pCoreServerShell)
		return;

	tagLeaveGame	sLeaveGame;	
	sLeaveGame.cProtocol = c2s_leavegame;   // ·¢ËÍÍùbishop Í¨Öª paysys
	sLeaveGame.cCmdType  = NORMAL_LEAVEGAME;

	tagLeaveGame2 lg2;
	
	lg2.ProtocolFamily = pf_normal;
	lg2.ProtocolID     = c2s_leavegame;	   // S3 Íæ¼ÒÀë¿ªÓÎÏ·
	lg2.cCmdType       = NORMAL_LEAVEGAME;
	lg2.nSelServer     = m_SerVerNo;
	// ±éÀúÍæ¼Ò£¨²»ÄÜ±éÀúÁ¬½Ó£¬ÒòÎªTimeout²¿·ÖµÄÍæ¼ÒÃ»ÓÐÁ¬½Ó£©
	for (int nIndex = 1; nIndex < MAX_PLAYER; ++nIndex)
	{
		if (m_pCoreServerShell->IsPlayerLoginTimeOut(nIndex))
		{//³¬Ê±µÇÂ½µÄ
			char szName[32]={0},szRoleName[64]={0};
			m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ACCOUNT, (unsigned int)szName, nIndex);
			//sLeaveGame.nExtPoint    = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTPOINTCHANGED, nIndex, 0); //»ñÈ¡Òª¿Û³ýµÄÀ©Õ¹µã
			//sLeaveGame.nGameExtTime = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTTIMECHANGED, nIndex, 0);
			sLeaveGame.nExtPoint    = m_pCoreServerShell->GetExpData(nIndex,0);
			sLeaveGame.nGameExtTime = m_pCoreServerShell->GetExpData(nIndex,1);
			m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)szRoleName, nIndex);

			//sprintf(sLeaveGame.nClientIp,"127.0.0.1");
			strcpy((char *)sLeaveGame.nClientIp, (char *)szRoleName);
			strcpy((char *)sLeaveGame.szAccountName, (char *)szName);

			if (m_pGatewayClient)
			   m_pGatewayClient->SendPackToServer(&sLeaveGame, sizeof(tagLeaveGame)); //Í¨Öªbishop

			strcpy((char *)lg2.szAccountName, (char *)szName);
			
			if (m_pTransferClient)//Í¨ÖªS3
				m_pTransferClient->SendPackToServer(&lg2, sizeof(tagLeaveGame2));
			//cout <<sLeaveGame.szAccountName << " ----------µÇÂ¼³¬Ê±leave game!-------------" << endl;
			printf("------(%s)µÇÂ¼³¬Ê±------- \n",sLeaveGame.szAccountName);
			//¶Ï¿ªÍøÂçºó²Å¿ÉÒÔÉ¾³ý¡¡ && m_pCoreServerShell->GetSaveStatus(nIndex)==SAVE_REQUEST
			if (m_pCoreServerShell) //ÒÑ¾­¶Ï¿ªÍøÂçµÄ
			    m_pCoreServerShell->RemovePlayerLoginTimeOut(nIndex);
		}
		else if (m_pCoreServerShell->IsCharacterQuiting(nIndex))
		{//³¬¹ýÈËÊý »òÕßÉèÖÃ ÍË³öµÄ

			if(!m_pCoreServerShell->IsShutdownClient(nIndex))
			{//¶Ï¿ª¿Í»§¶Ë
				//cout << " ----------ÍË³öÓÎÏ·,·µ»Ø!-------------" << endl; 
				continue;
			}
            //ÒÑ¾­¶Ï¿ªÍøÂçµÄ
			if (m_pCoreServerShell->GetSaveStatus(nIndex)!=SAVE_REQUEST)
			{
				continue;
			}
			/*int j;
			j = FindSame(lnID); // ²éÕÒÏàÍ¬
			if (j)
			{
				int nIndex = m_pGameStatus[j].nPlayerIndex;
			} */

			//if(m_pCoreServerShell->IsCharacterLiXian(nIndex) == 0 || m_pCoreServerShell->IsCharacterLiXian(nIndex) == 1)  //²»ÊÇÀëÏß¹Ò»úµÄ ²ÅÂíÉÏ´æµµ
			//	SavePlayerData(nIndex,true); //Á¢¼´´æµµ

//			if (!SavePlayerData(nIndex))
//				continue;

		    //if (m_pCoreServerShell->GetClientNetConnectIdx(nIndex)==-1) //·þÎñÆ÷ÒÑ¾­¶ÏÏßµÄ ´¦Àí
			//	return;

			if(m_pCoreServerShell->IsCharacterLiXian(nIndex) == 0 || m_pCoreServerShell->IsCharacterLiXian(nIndex) == 1)
			{//²»ÊÇÍÐ¹ÜµÄ
				char	szName[32]={0};
				m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ACCOUNT, (unsigned int)szName, nIndex);
				
				sLeaveGame.nExtPoint    = m_pCoreServerShell->GetExpData(nIndex,0);
				//m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTPOINTCHANGED, nIndex, 0); // »ñÈ¡Òª¿Û³ýµÄÀ©Õ¹µã
				sLeaveGame.nGameExtTime = m_pCoreServerShell->GetExpData(nIndex,1);
				//m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTTIMECHANGED, nIndex, 0);
				sprintf(sLeaveGame.nClientIp,"127.0.0.1");
				strcpy((char *)sLeaveGame.szAccountName, (char *)szName);
				if (m_pGatewayClient)
				   m_pGatewayClient->SendPackToServer(&sLeaveGame, sizeof(tagLeaveGame));
				
				strcpy((char *)lg2.szAccountName, (char *)szName);
				
				if (m_pTransferClient)
				   m_pTransferClient->SendPackToServer(&lg2, sizeof(tagLeaveGame2));

				if (m_pCoreServerShell->IsCharacterLiXian(nIndex) == 1)
				{//ÉèÖÃÄ£Ê½Îª2
					m_pCoreServerShell->SetCharacterLiXian(nIndex,2);
					printf("------(%s)ÉèÖÃ·þÎñÆ÷ÀëÏß³É¹¦!------- \n",sLeaveGame.szAccountName);
				}
			}
/*
			m_pGameStatus[lnID].nExchangeStatus = enumExchangeBegin;
			m_pGameStatus[lnID].nGameStatus     = enumPlayerBegin;
			m_pGameStatus[lnID].nPlayerIndex    = 0;
*/         
			m_pCoreServerShell->RemoveQuitingPlayer(nIndex);   //É¾³ýÍæ¼ÒÁ¢¼´´æµµ
		}

	}
}
//»ñÈ¡ÍøÂç×´Ì¬
int KSwordOnLineSever::GetNetStatus(const unsigned long lnID)
{
	if (lnID >= m_nMaxGameStaus)  //ÍøÂçºÅ´óÓÚ×î´óÈËÊý ÉèÖÃÎª²»Á´½Ó×´Ì¬
		return enumNetUnconnect;

	return m_pGameStatus[lnID].nNetStatus;
}
//ÉèÖÃÍøÂç×´Ì¬
void KSwordOnLineSever::SetNetStatus(const unsigned long lnID, NetStatus nStatus,bool isSave)
{
	//if (lnID >= m_nMaxGameStaus)
	//	return;
	//const char* pInfo = m_pServer->GetClientInfo(lnID);
	 
	if (nStatus == enumNetUnconnect)
	{//ÉèÖÃ¶Ï¿ªÁ¬½Ó×´Ì¬
		int j;
		    j = FindSame(lnID); // ²éÕÒÏàÍ¬
		if (j)
		{
		   int nIndex = m_pGameStatus[j].nPlayerIndex;
		   if (m_pGameStatus[j].nGameStatus == enumPlayerPlaying\
			|| (m_pGameStatus[j].nGameStatus == enumPlayerExchangingServer\
			&& m_pGameStatus[j].nExchangeStatus != enumExchangeCleaning))	// ¿ç·þÎñÆ÷Ê±×Ô¼º´¦Àí
		   { //ÓÎÏ·ÖÐ »ò ×ª»»µØÍ¼ ¿ªÊ¼´æÒ»´Îµ²
			  
			   if (m_pCoreServerShell->GetClientNetConnectIdx(nIndex)>-1)
			   {//È·±£ÔÚ É¾³ý½ÚµãÇ°¶ÏÏßÇ°Éú³É ´æµµ°ü
				   m_pCoreServerShell->CheckSerMaps(nIndex,TRUE);
				   m_pCoreServerShell->SetSaveStatus(nIndex,SAVE_IDLE);
				   if (isSave)
			          m_pCoreServerShell->SavePlayerData(nIndex);	          //Éú³É´æµµ°ü,ÂíÉÏ´æµµ
				   //printf("------ÍË³öÓÎÏ· ´æµµ³É¹¦(%d)------\n",nIndex);
			   }
			   else
			   {
				   printf("------ÍË³öÓÎÏ·,ÒÑ¾­¶ÏÏß,²»×ö´æµµ´¦Àí(%d)------\n",nIndex);
			   }
		   }  
		   else
		   {   //µÇÂ½Ê§°Ü
			   m_pCoreServerShell->PreparePlayerForLoginFailed(nIndex);
		   }

			   m_FreeIdxNetStatus.Insert(j);        //É¾³ýÁ´±í
			   m_UseIdxNetStatus.Remove(j);

			   m_pGameStatus[j].nGameStatus = enumPlayerNone;             //enumPlayerBegin;
			   m_pGameStatus[j].nPlayerIndex = 0;
			   m_pGameStatus[j].nExchangeStatus = enumExchangeCleaning;   //ÉèÖÃ¿ªÊ¼×ª·þ
			   m_pGameStatus[j].nReplyPingTime = 0;
			   m_pGameStatus[j].nSendPingTime = 0;
			   m_pGameStatus[j].nNetidx=-1;
			   m_pGameStatus[j].nNetStatus = nStatus;
			   m_pGameStatus[j].nErrorLoopCount = 0;
			   m_pGameStatus[j].nPingLoopCount  = 0;

			   //printf("------ÍøÂç(%u)É¾³ý½Úµã³É¹¦(%d),×´Ì¬:%d------\n",lnID,j,m_pGameStatus[j].nNetStatus);

		}  

		/*
		m_pGameStatus[lnID].nGameStatus = enumPlayerBegin;
		m_pGameStatus[lnID].nPlayerIndex = 0;
		m_pGameStatus[lnID].nExchangeStatus = enumExchangeBegin;   //ÉèÖÃ¿ªÊ¼×ª·þ
		m_pGameStatus[lnID].nReplyPingTime = 0;
		m_pGameStatus[lnID].nSendPingTime = 0;
		*/

	}

	if (nStatus == enumNetConnected)
	{//ÉèÖÃÍøÂçÁ´½Ó×´Ì¬
	   int i;
		i = FindFree();
		if (i>0)
		{
			m_FreeIdxNetStatus.Remove(i);
			m_UseIdxNetStatus.Insert(i);

			m_pGameStatus[i].nGameStatus = enumPlayerBegin;
			m_pGameStatus[i].nPlayerIndex = 0;
			m_pGameStatus[i].nExchangeStatus = enumExchangeBegin;//enumExchangeBegin;
			m_pGameStatus[i].nReplyPingTime = 0;
		    m_pGameStatus[i].nSendPingTime  = m_nGameLoop;
			m_pGameStatus[i].nNetidx=lnID;
			m_pGameStatus[i].nNetStatus = nStatus;
			m_pGameStatus[i].nErrorLoopCount = 0;
			m_pGameStatus[i].nPingLoopCount  = 0;
			//printf("------ÍøÂç(%u)Ôö¼Ó½Úµã(%d),×´Ì¬:%d ³¬Ê±¼ÆÊ±:%u------\n",lnID,i,m_pGameStatus[i].nNetStatus,m_pGameStatus[i].nSendPingTime);
		}
		/*
		m_pGameStatus[lnID].nGameStatus = enumPlayerBegin;
		m_pGameStatus[lnID].nPlayerIndex = 0;
		m_pGameStatus[lnID].nExchangeStatus = enumExchangeBegin;
		m_pGameStatus[lnID].nReplyPingTime = 0;
		m_pGameStatus[lnID].nSendPingTime = 0;
		*/
	}

	//m_pGameStatus[lnID].nNetStatus = nStatus;		
}
//µÇÂ½Ð­Òé

//inline const char* _ip2a(DWORD ip) { in_addr ia; ia.s_addr = ip; return inet_ntoa(ia); }
//inline DWORD _a2ip(const char* cp) { return inet_addr(cp); }
int KSwordOnLineSever::ProcessLoginProtocol(const unsigned long lnID, const char* pData, size_t dataLength)
{
	const char* pBuffer = pData;
//ÊÕµ½¿Í»§¶ËµÄÇëÇó
	if (*pBuffer != c2s__loginfs_kfjghtueodnchsf)  //Èç¹ûÐ­Òé±êÊ¾´íÎó¾Í·µ»Ø0
	{
		printf("----------µÇÂ½Ê§°Ü·µ»ØA-------- \n");
		m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);  //¹Ø±Õ¿Í»§¶Ë
		return 0;
	}
	else
	{
		//---------------IP¹ýÂËÉèÖÃ-----------------------------------------------------
			/*char szDesMsg[32];
		sprintf(szDesMsg,"%s",m_pServer->GetClientInfo(lnID)); //»ñÈ¡Á´½ÓµÄÍøÂçºÅµÄ IP
        
		//int pRect[4];
        //sscanf(szDesMsg, "%d.%d.%d", &pRect[0], &pRect[1], &pRect[2]);
        //if (pRect[0]!=192 && pRect[1]!=168)
	   if ((pRect[0]==192 && pRect[1]==168) || (pRect[0]==127 && pRect[1]==0 && pRect[2]==0))
		{

		}
	    else
		{ 
			m_pServer->ShutdownClient(lnID);  //¹Ø±Õ¿Í»§¶Ë
		    //--SetRunningStatus(FALSE);---          //ÉèÖÃGSÍ£Ö¹ÔËÐÐ
			return 0;
		} */
		//---------------------------------------------------------------------------
		printf("----------½ÓÊÜ¿Í»§¶Ë·¢°ü³É¹¦:c2s__loginfs_kfjghtueodnchsf--------\n");
        //Õâ¸öÍøÂçºÅÎÞÏÞÔö´ó
		tagLogicLogin *pLL = (tagLogicLogin *)pData;

		if  (pLL->cProtocol!=c2s__loginfs_kfjghtueodnchsf)
		{
			printf("----------µÇÂ½Ê§°Ü·µ»ØB--------\n");
			m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);  //¹Ø±Õ¿Í»§¶Ë
		    return 0;
		}

		//»ñÈ¡Íæ¼ÒµÄ nUseIdx Á´±íºÅ	 playerid
		int nIdx = m_pCoreServerShell->AttachPlayer(m_pGameStatus[lnID].nNetidx,&pLL->guid);   //lnID ÎªÍøÂçºÅ	½«ÍøÂçÁ´½ÓºÅ¸³Öµ¸øÈËÎï

		if (nIdx)
		{
		   printf("¡ï¡ï¡ï¡ï¡ï½ÓÊÜ¿Í»§¶Ë(%d)µÇÂ½ÇëÇó³É¹¦¡ï¡ï¡ï¡ï¡ï\n",m_pGameStatus[lnID].nNetidx);
           return nIdx;
		}
		else
		{
			// ·Ç·¨µÄÍæ¼Ò£¬¸ÃÔõÃ´´¦ÀíÔõÃ´´¦Àí
			/*
			buf.Format("{%08X-%04X-%04x-%02X%02X-%02X%02X%02X%02X%02X%02X}"
			, guid.Data1
			, guid.Data2
			, guid.Data3
			, guid.Data4[0], guid.Data4[1]
			, guid.Data4[2], guid.Data4[3], guid.Data4[4], guid.Data4[5]
       ,      guid.Data4[6], guid.Data4[7]);
			
			*/
			printf("·Ç·¨µÇÂ½(%d)£¬·Ç·¨µÇÂ½C<%08X-%04X-%04x-%02X%02X-%02X%02X%02X%02X%02X%02X> \n"\
				,lnID,pLL->guid.Data1,pLL->guid.Data2\
				,pLL->guid.Data3,pLL->guid.Data4[0],pLL->guid.Data4[1]\
				,pLL->guid.Data4[2],pLL->guid.Data4[3],pLL->guid.Data4[4],pLL->guid.Data4[5]\
                ,pLL->guid.Data4[6], pLL->guid.Data4[7]);

			m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);  //¹Ø±Õ¿Í»§¶Ë
			return 0;
		}
	}
	return 0;	
}
//ÓÐÎÊÌâ ÈÝÒ×¿¨ºÅ ¾ÍÔÚÕâÀïÁË
BOOL KSwordOnLineSever::SendGameDataToClient(const unsigned long lnID, const int nPlayerIndex)
{
	BOOL bRet = FALSE;
//#ifndef _STANDALONE
	_ASSERT(m_pServer);

	int				nStep = 0;
	unsigned int	nParam = 0;
	int				bSyncEnd = 0;

	m_pServer->PreparePackSink(); //Í¨Öª·þÎñÆ÷×¼±¸·¢°ü

	while(true)  //Ñ­»·Í¬²½ Íæ¼ÒµÇÂ½ÐÅÏ¢
	{
		bSyncEnd = (nStep == STEP_SYNC_END);   //Èç¹û nStep == STEP_SYNC_END,¾Í½« TRUE ¸³Öµ¸ø bSyncEnd, ·ñÔò FALSE;
		//¼ÓÔØÍæ¼ÒÊý¾Ý ¶Áµµ
		bRet = m_pCoreServerShell->PlayerDbLoading(nPlayerIndex, bSyncEnd, nStep, nParam);

		if (!bRet)
		{
			printf(".....¶Áµµ Ê§°Ü.....\n");	//[wxb 2003-7-28]
			break;
		}

		//if (nParam!=0)
		//	continue;

		if (bSyncEnd)
		{	//Í¨Öª¿Í»§¶Ë Í¬²½Íê±Ï		
			m_pGameStatus[lnID].nGameStatus = enumPlayerSyncEnd;
			printf("-----¶ÁµµÍ¬²½Íê±Ï(%d),Í¨Öª¿Í»§¶ËÍøÂçºÅ(%d)µÇÂ½----- \n",bSyncEnd,m_pGameStatus[lnID].nNetidx);
			CLIENT_SYNC_END	SyncEnd;
			//BYTE	SyncEnd;
			SyncEnd.ProtocolType = s2c_syncclientend;  //(BYTE)
			SyncEnd.m_IsLogin    = TRUE;
			SyncEnd.m_clientKey  = 9473290;
			g_SetRootPath(NULL);
			g_SetFilePath("\\");
			KIniFile iniFile;
			if (iniFile.Load("ServerCfg.ini"))
			{
				int tempInt;
				iniFile.GetInteger("GameServer","ClientKey",8570294,&tempInt);
				SyncEnd.m_clientKey = tempInt;
				iniFile.Clear();
			}

#ifndef _STANDALONE
			if (FAILED(m_pServer->PackDataToClient(m_pGameStatus[lnID].nNetidx,&SyncEnd, sizeof(CLIENT_SYNC_END)))) //(BYTE*)
#else
			if (!SUCCEEDED(m_pServer->PackDataToClient(m_pGameStatus[lnID].nNetidx,&SyncEnd, sizeof(CLIENT_SYNC_END))))//(BYTE*)
#endif
			{
				bRet = FALSE;
				break;
			}

			break;
		}
	}
//ÊÇ·ñÄÜÁ´½ÓÉÏ¿Í»§¶Ë,·ñÔò¾ÍÊÇÊ§°Ü!
#ifdef WIN32
	if (FAILED(m_pServer->SendPackToClient(m_pGameStatus[lnID].nNetidx)))
	{
		bRet = FALSE;
	}
#else
	if (m_pServer->SendPackToClient(m_pGameStatus[lnID].nNetidx) <= 0)
		bRet = FALSE;
#endif

	return bRet;
}
//ÊÇ·ñÒÑ¾­Í¬²½Íê¿Í»§¶Ë
BOOL KSwordOnLineSever::ProcessSyncReplyProtocol(const unsigned long lnID, const char* pData, size_t dataLength)
{
	/*const char*	pBuffer = pData;

	if (*pBuffer != c2s_syncclientend)
	{
		return FALSE;
	}
	else
	{
		return TRUE;
	}*/
	if  (!pData) return false;
	CLIENT_SYNC_END *nCurInfo = (CLIENT_SYNC_END *)pData;
	g_SetRootPath(NULL);
	g_SetFilePath("\\");
	KIniFile iniFile;
	int      tempInt;
	if (iniFile.Load("ServerCfg.ini"))
	{
		iniFile.GetInteger("GameServer","ClientKey",7547593,&tempInt);
		iniFile.Clear();
	}

	if (nCurInfo->ProtocolType !=c2s_syncclientend || !nCurInfo->m_IsLogin ||nCurInfo->m_clientKey!=tempInt) return false;

	return true;	
}


void KSwordOnLineSever::ExitAllPlayerIngame()
{

#ifndef __linux	
			Sleep(4000);
#else
			Sleep(4);
#endif

	int lnID;
	for (lnID = 0; lnID < m_nMaxGameStaus; ++lnID)
	{
		int nIndex = m_pGameStatus[lnID].nPlayerIndex;
		if (nIndex <= 0 || nIndex>=MAX_PLAYER)
			continue; 

		if (!m_pCoreServerShell->IsShutdownClientNoserver(nIndex))
		{//¸ºÔð¶Ï¿ª¿Í»§¶Ë
			continue;
		}
	}
}
//BiSHOPÌßµôËùÓÐÍæ¼Ò
void KSwordOnLineSever::ExitAllPlayer()
{

	
#ifndef __linux			//SwitchToThread();
			Sleep(4000);
#else
		     //usleep(4000000);
			Sleep(4);
#endif

	int lnID;
	/*const char*	pData;
	unsigned int	uSize;
	tagLeaveGame	sLeaveGame;	

	sLeaveGame.cProtocol = c2s_leavegame;
	sLeaveGame.cCmdType = NORMAL_LEAVEGAME;
	*/
	for (lnID = 0; lnID < m_nMaxGameStaus; ++lnID)
	{
		int nIndex = m_pGameStatus[lnID].nPlayerIndex;
		if (nIndex <= 0 || nIndex>=MAX_PLAYER)
			continue; 

		if  (m_pCoreServerShell)
			m_pCoreServerShell->IsShutdownClientOutserver(nIndex);

		/*if (!m_pCoreServerShell->IsShutdownClient(nIndex))
		{//¸ºÔð¶Ï¿ª¿Í»§¶Ë
			continue;
		}*/
	/*
		BOOL bSave = m_pCoreServerShell->SavePlayerData(nIndex);  //´æÅÌ

		int nCount = 0;

		while(bSave)
		{
			nCount++;
			pData = (const char *)m_pDatabaseClient->GetPackFromServer(uSize);
			if (pData && uSize)
			{
				DatabaseMessageProcess(pData, uSize);
				if (m_pCoreServerShell->GetSaveStatus(nIndex) == SAVE_IDLE)
					break;
			}

			if (nCount > 1000)
				break;
#ifdef WIN32
			Sleep(1);
#else
			Sleep(1);
#endif
		}

		m_pCoreServerShell->ClientDisconnect(nIndex,TRUE);          //ÉèÖÃÀë¿ªÓÎÏ·×´Ì¬--¶Ï¿ª¿Í»§¶ËÁ´½Ó

		if (m_pCoreServerShell->IsCharacterQuiting(nIndex))
		{
			char	szName[32];
	
			m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ACCOUNT, (unsigned int)szName, nIndex);
			sLeaveGame.nExtPoint    = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTPOINTCHANGED, nIndex, 0);  //»ñÈ¡Òª¿Û³ýµÄÀ©Õ¹µã
            sLeaveGame.nGameExtTime = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_EXTTIMECHANGED, nIndex, 0);

			  sprintf(sLeaveGame.nClientIp,"127.0.0.1");

			//printf("------ÍË³öÓÎÏ·µÄIP:%s,×ÜÈËÊý:%d\n",nClientInfo.nClientIp,nClientInfo.nNumPlayer);
			strcpy((char *)sLeaveGame.szAccountName, (char *)szName);
			    m_pGatewayClient->SendPackToServer(&sLeaveGame, sizeof(tagLeaveGame)); //56732

			if (m_pTransferClient) //·¢Íùbishop
				m_pTransferClient->SendPackToServer(&sLeaveGame, sizeof(tagLeaveGame));

			m_pCoreServerShell->RemoveQuitingPlayer(nIndex);
		} 
		

#ifdef WIN32
			Sleep(1);
#else

			Sleep(1);
#endif */
	}
}
//ÉèÖÃGSÊÇ·ñÔËÐÐ
void KSwordOnLineSever::SetRunningStatus(BOOL bStatus)
{
	m_bIsRunning = bStatus;
}

void KSwordOnLineSever::SendCilentPackData()
{
	LONGLONG nMes = 1000;
	if (nMes * m_nGameLoop <= m_Timer.GetElapse()*GAME_FPS) //1Ãë18Ö¡  Ê±¼äÒç³ö 
	{	
	   int lnID = m_nGameLoop%m_nMaxGameStaus;	  //Ëæ»úµÄ
	   int nIndex = m_pGameStatus[lnID].nPlayerIndex; //
	   if (nIndex > 0 && nIndex < MAX_PLAYER && m_pGameStatus[lnID].nGameStatus == enumPlayerPlaying)
	   {//ÔÚÏßµÄ¾Í¼ì²â
#define	defMAX_PING_TIMEOUT		60*GAME_FPS
#define	defMAX_PING_INTERVAL	5*GAME_FPS
#define defMAX_CONNECTION_TIMEOUT (15 * GAME_FPS) 

		  if (m_pGameStatus[lnID].nReplyPingTime != 0)	//ÉÏ´Î·µ»ØµÄpingÊ±¼ä
		  {//¿Í»§¶ËÓÐÏìÓ¦µÄ  
			 if (m_nGameLoop - m_pGameStatus[lnID].nSendPingTime > defMAX_PING_INTERVAL)
			 { 
				PingClient(lnID);
				//printf("Send ping to clienta.... \n");
			 } 
		  }  
		  else if (m_pGameStatus[lnID].nReplyPingTime==0 && m_pGameStatus[lnID].nPingLoopCount<m_pingCount && (m_nGameLoop - m_pGameStatus[lnID].nSendPingTime > defMAX_PING_TIMEOUT))
		  {//½ÓÊÜ¿Í»§¶Ë·µ»ØµÄÊ±¼ä²î ¿Í»§¶ËÃ»ÏìÓ¦µÄ
			  m_pGameStatus[lnID].nPingLoopCount++;
			  printf("Send ping to clientb.... \n");
			  PingClient(lnID);	//Ë÷Òý
		  }
		  else if (m_nGameLoop - m_pGameStatus[lnID].nSendPingTime > defMAX_PING_TIMEOUT)
		  {//³¬Ê±ÁË
			  printf("no response from client(%d, %d), ShutdownClient it...\n", m_nGameLoop, m_pGameStatus[lnID].nSendPingTime);
             //ÉèÖÃ¶Ï¿ªÁ´½Ó
			  m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);                            //¹Ø±Õ¿Í»§¶Ë
		  } 
	   } 
	   else if (m_pGameStatus[lnID].nNetidx >0 && m_pGameStatus[lnID].nNetStatus == enumNetConnected && m_pGameStatus[lnID].nGameStatus == enumPlayerBegin)
	   {
		   // ¼ì²éÒÑ¾­Á¬½Óµ«ÊÇ»¹Ã»ÓÐ·¢µÇÂ¼Ð­ÒéµÄÍæ¼Ò
		   if (m_nGameLoop - m_pGameStatus[lnID].nSendPingTime > defMAX_CONNECTION_TIMEOUT)
		   {
			   printf("Shutdown non-login client[%u] for timeout\n",m_pGameStatus[lnID].nNetidx);
			   m_pServer->ShutdownClient(m_pGameStatus[lnID].nNetidx);
			   if (m_pGameStatus[lnID].nNetidx>0)
			      SetNetStatus(m_pGameStatus[lnID].nNetidx,enumNetUnconnect,false);             //ÉèÖÃ¶Ï¿ªÁ´½Ó
		   }
	   }
	   if (m_nGameLoop & 0x01)  //Ã¿Ö¡¶¼ÏòËùÓÐ¿Í»§¶Ë·¢°ü  
	   { 
		   m_pServer->SendPackToClient(-1);  //ÏòËùÓÐÁ´½Ó·¢°ü
	   } 
	}
}

void KSwordOnLineSever::SetClientConnetNet(int nIsThis)
{
	switch(nIsThis)
	{
	case 9:
		{//npc×´Ì¬
			if (m_pCoreServerShell)
				m_pCoreServerShell->ExeCoreNpcState();
		}
		break;
	case 1:
		{//Íø¹Ø¼à¿Ø
		  if (m_pGatewayClient)
		  {
			if (FAILED(m_pGatewayClient->FsGameServerConnectTo(m_szGatewayIP,m_nGatewayPort)))
			{  
				cout << " - Íø¹Ø·þÎñÆ÷ÖØÐÂÁ´½Ó Ê§°Ü..!" << endl;
			}  
			else
			{ 	
				g_pSOServer.SetClientNet(GATWSTATE,TRUE);
				cout << " - Íø¹Ø¿â·þÎñÆ÷ÖØÐÂÁ´½Ó OK..!" << endl;
			}
		  }
		}
		break;
	case 2:
		{//Êý¾Ý¿â¼à¿Ø
		  if (m_pDatabaseClient)
		  {
			if (FAILED(m_pDatabaseClient->FsGameServerConnectTo(m_szDatabaseIP,m_nDatabasePort)))
			{  
				cout << " - Êý¾Ý¿â·þÎñÆ÷ÖØÐÂÁ´½Ó Ê§°Ü..!" << endl;
			}  
			else
			{ 	
				g_pSOServer.SetClientNet(DATASTATE,TRUE);
				cout << " - Êý¾Ý¿â·þÎñÆ÷ÖØÐÂÁ´½Ó OK..!" << endl;
			} 
		  }
		}
		break;
	case 3:
		{
		  if (m_pChatClient)
		  {
		    if (FAILED(m_pChatClient->FsGameServerConnectTo(m_szChatIP,m_nChatPort)))
			{  
				cout << " - ÁÄÌì·þÎñÆ÷ÖØÐÂÁ´½Ó Ê§°Ü..!" << endl;
			}  
			else
			{ 	
				g_pSOServer.SetClientNet(CHATSTATE,TRUE);
				cout << " - ÁÄÌì·þÎñÆ÷ÖØÐÂÁ´½Ó OK..!" << endl;
			}
		  }
		}
		break;
	case 4:
		{
		  if (m_pTongClient)
		  {
			if (FAILED(m_pTongClient->FsGameServerConnectTo(m_szTongIP,m_nTongPort)))
			{  
				cout << " - °ï»á·þÎñÆ÷ÖØÐÂÁ´½Ó Ê§°Ü..!" << endl;
			}  
			else
			{ 	
				g_pSOServer.SetClientNet(TONGSTATE,TRUE);
				cout << " - °ï»á·þÎñÆ÷ÖØÐÂÁ´½Ó OK..!" << endl;
			} 
		  }
		}
		break;
	case 5:
		{
		  if (m_pTransferClient)
		  {
			if (FAILED(m_pTransferClient->FsGameServerConnectTo(m_szTransferIP,m_nTransferPort)))
			{  
				cout << " - ¿ç·þ·þÎñÆ÷ÖØÐÂÁ´½Ó Ê§°Ü..!" << endl;
			}  
			else
			{ 	
				g_pSOServer.SetClientNet(TRANSTATE,TRUE);
				cout << " - ¿ç·þ·þÎñÆ÷ÖØÐÂÁ´½Ó OK..!" << endl;
			}
		  }
		}
		break;
	case 6:
		{//ÈÎÎñ¡¡»ò¡¡´æµµ¡¡¼à¿Ø
			if (m_pCoreServerShell)
				m_pCoreServerShell->ExeCoreTask();
         //¼ì²âÏÞÊ±ÎïÆ·
			if (m_pCoreServerShell)
				m_pCoreServerShell->CheckItemTime();
		}
		break;
	case 7:
		{//ÅÄÂôÐÐ Í¬²½
			if (m_pCoreServerShell)
				m_pCoreServerShell->SynCoreItemData();
		}
		break;
	case 8:
		{
//²éÑ¯ÅÅÃû
			if (m_mapClientNet[DATASTATE] && m_pDatabaseClient)
			{
					TProcessData ProcessData;
					ProcessData.nProtoId = c2s_gamestatistic;
					ProcessData.nDataLen = 1;
					ProcessData.ulIdentity = -1;
					ProcessData.pDataBuffer[0] = 0;
					m_pDatabaseClient->SendPackToServer(&ProcessData, sizeof(TProcessData));
			}
		}
		break;
	default:
		break;
	}
}

void KSwordOnLineSever::SetS3RelayRunStatus(BOOL bStatus,int nType)
{
	/*if (nType==1)
	   m_ChatState = bStatus;
	else if (nType==2)
	   m_TongState = bStatus;
	else if (nType==3)
	   m_TranState = bStatus;
	else if (nType==4)
	   m_pDataState = bStatus;
	else if (nType==5)
	   m_GatewayState = bStatus;*/
	switch(nType)
	{
	case 1:
		{
	          m_ChatState = bStatus;
		}
		 break;
	case 2:
		 {
			  m_TongState = bStatus;
		 }
		 break;
	case 3:
		 {
			 m_TranState = bStatus;
		 }
		 break;
	case 4:
		 {
			  m_pDataState = bStatus;
		 }
		 break;
	case 5:
		 {
			 m_GatewayState = bStatus;
		 }
		 break;
	default:
		break;
	}
}

BOOL KSwordOnLineSever::ConformAskWay(const void* pData, int nSize, DWORD *lpnNetID)
{
	BOOL	bRet = FALSE;
	RELAY_ASKWAY_DATA*	pRAD = (RELAY_ASKWAY_DATA *)pData;

	// TODO: ¼ì²éÊÇ·ñÑ°Â·ÕýÈ·
	int i;
	int nMethod = pRAD->seekMethod;
	switch(nMethod)
	{
	case rm_role_id:
	case rm_account_id:
		{
			DWORD	dwNameID;
			DWORD	lnID;
			LPDWORD	lpdwPTR = (DWORD *)(((char *)pData) + sizeof(RELAY_ASKWAY_DATA) + pRAD->wMethodDataLength - sizeof(DWORD)*2);
			dwNameID = *lpdwPTR;
			lnID = *(lpdwPTR + 1); //ÍøÂçID
			 i=FindSame(lnID);

			if (/*lnID >= m_nMaxGameStaus*/i >= m_nMaxGameStaus || m_pGameStatus[i].nPlayerIndex <= 0 || m_pGameStatus[i].nPlayerIndex >= MAX_PLAYER)
			{
				bRet = FALSE;
				break;
			}
			DWORD	dwNameIDbyIndex = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ID, m_pGameStatus[i].nPlayerIndex, 0);
			if (dwNameIDbyIndex != dwNameID)
			{
				bRet = FALSE;
				break;
			}
			*lpnNetID = lnID;
			bRet = TRUE;
		}
		break;
	case rm_map_id:
		{
			/*DWORD	dwMapID;
			printf("---ConformAskWay:rm_map_id----\n");
			dwMapID = *(DWORD *)(((char *)pData) + sizeof(RELAY_ASKWAY_DATA));

			if (0)	// TODO:check mapid here
			{
				bRet = FALSE;
				break;
			}*/
			printf("---ConformAskWay:rm_map_id----\n");
			bRet = TRUE;
		}																										  		break;

	}
	return bRet;
}
/////////////////////////////////»ñÈ¡±¾µØIPÉèÖÃ////////////////////////////////
int KSwordOnLineSever::GetLocalIpAddress(DWORD *pIntranetAddr, DWORD *pInternetAddr)
{
#ifndef _STANDALONE
	return gGetMacAndIPAddress(NULL, pIntranetAddr, NULL, pInternetAddr);
#else
	char	szIp[16];
	DWORD	dwAddr1, dwAddr2;
	dwAddr1 = dwAddr2 = 0;
	WSADATA wsData;  
    ::WSAStartup(MAKEWORD(2,2), &wsData);

	char strHost[MAX_PATH];
	strHost[0] = 0;
	if(SOCKET_ERROR != gethostname(strHost, sizeof(strHost)))  //»ñÈ¡Ö÷»úµÄÃû³Æ
	{
		struct hostent* hp;
		hp = gethostbyname(strHost);  //»ñÈ¡Ö÷»úµÄip
	
#ifndef WIN32  //LÏµÍ³
		if (!hp || hp && hp->h_addr_list[0] && *(unsigned long *)hp->h_addr_list[0] == 0x0100007f)
		{
			int sock;
			struct ifreq ifr;
			char* ip = NULL;
			int err = -1;

			sock = socket(PF_INET, SOCK_DGRAM, IPPROTO_IP);
			if (sock >= 0)
			{
				strcpy(ifr.ifr_name, "eth0");
				ifr.ifr_addr.sa_family = AF_INET;
				err = ioctl(sock, SIOCGIFADDR, &ifr);
				if (err == 0) 
				{
					dwAddr1 = (((struct sockaddr_in*) &(ifr.ifr_addr))->sin_addr).s_addr;
					ip =  inet_ntoa(((struct sockaddr_in*) &(ifr.ifr_addr))->sin_addr);
//					printf("IP-addr1: %s\n", ip); 
				}

				strcpy(ifr.ifr_name, "eth1");
				ifr.ifr_addr.sa_family = AF_INET;

				err = ioctl(sock, SIOCGIFADDR, &ifr);
				if (err == 0) {
					dwAddr2 = (((struct sockaddr_in*) &(ifr.ifr_addr))->sin_addr).s_addr;
					ip =  inet_ntoa(((struct sockaddr_in*) &(ifr.ifr_addr))->sin_addr);
//					printf("IP-addr2: %s\n", ip);
				}
				else
					dwAddr2 = dwAddr1;
				closesocket(sock);
			}
		}
		else
#endif
		{

		  if(hp && hp->h_addr_list[0])
		  { 
			dwAddr1 = *(unsigned long *)hp->h_addr_list[0];
/*
			*((char *)&dwAddr1) = hp->h_addr_list[0][3];
			*((char *)&dwAddr1 + 1) = hp->h_addr_list[0][2];
			*((char *)&dwAddr1 + 2) = hp->h_addr_list[0][1];
			*((char *)&dwAddr1 + 3) = hp->h_addr_list[0][0];*/
//			dwAddr1 = *(unsigned long *)hp->h_addr_list[0];/
			if (!dwAddr1)
				dwAddr1 = *(unsigned long *)hp->h_addr_list[1];
		  } 

		  if(hp && hp->h_addr_list[1])
		  {
			dwAddr2 = *(unsigned long *)hp->h_addr_list[1];
		
/*
			*((char *)&dwAddr2) = hp->h_addr_list[1][3];
			*((char *)&dwAddr2 + 1) = hp->h_addr_list[1][2];
			*((char *)&dwAddr2 + 2) = hp->h_addr_list[1][1];
			*((char *)&dwAddr2 + 3) = hp->h_addr_list[1][0];*/
//			dwAddr2 = *(unsigned long *)hp->h_addr_list[1];

			if (!dwAddr2)
				dwAddr2 = *(unsigned long *)hp->h_addr_list[0];
			
		  }
		  else
		  {
			dwAddr2 = dwAddr1;
		  }
		}
	}

	if ((dwAddr1 & 0x0000FFFF) == 0x0000a8c0)	// intranet
	{
		*pIntranetAddr = dwAddr1;
		*pInternetAddr = dwAddr2;
	}
	else
	{
		*pIntranetAddr = dwAddr2;
		*pInternetAddr = dwAddr1;
	}
    ::WSACleanup();

	return TRUE;//gGetMacAndIPAddress(NULL,pIntranetAddr,NULL,pInternetAddr);
#endif
}

//////////////////////////////////////////////////////////////////////////////////////////////////
void KSwordOnLineSever::TransferAskWayMessageProcess(const char *pData, size_t dataLength)
{
	char	szChar[1024]={0};

	EXTEND_HEADER	*pEH = (EXTEND_HEADER *)pData;
	RELAY_ASKWAY_DATA	*pRAD = (RELAY_ASKWAY_DATA *)pData;
	
	RELAY_DATA	*pRD = (RELAY_DATA *)szChar;
	pRD->ProtocolFamily = pEH->ProtocolFamily;
	pRD->nToIP      = pRAD->nFromIP;  
	pRD->nToRelayID = pRAD->nFromRelayID;
	pRD->nFromIP = 0;
	pRD->nFromRelayID = 0;
	pRD->routeDateLength = pRAD->routeDateLength;
	
	DWORD	lnID;
	// TODO: check routeDataLength
	if (!ConformAskWay(pRAD, dataLength, &lnID))
	{//×ª·þ Ê§°Ü
		pRD->ProtocolID = relay_s2c_loseway;
		memcpy(szChar + sizeof(RELAY_DATA), pData, dataLength);
		//printf("---GS -->S3:relay_s2c_loseway--\n");
		m_pTransferClient->SendPackToServer(pRD, sizeof(RELAY_DATA) + dataLength);
	}
	else
	{
		pRD->ProtocolID = relay_c2c_data;
		switch(pRAD->seekMethod)
		{
		case rm_map_id:
			pRD->nToIP = mTransSerVerNo;
			pRD->nToRelayID = 0;
			memcpy(szChar + sizeof(RELAY_DATA), pData + sizeof(RELAY_ASKWAY_DATA) + pRAD->wMethodDataLength, pRD->routeDateLength);
			if (m_pTransferClient)
			    m_pTransferClient->SendPackToServer(pRD, sizeof(RELAY_DATA) + pRD->routeDateLength);
			//printf("---GS -->S3:relay_c2c_data---\n");
			break;
		case rm_role_id:
		case rm_account_id:
			{   
				int i=FindSame(lnID);
		        if (!i)
		           return;
				if (pf_normal != *(pData + sizeof(RELAY_ASKWAY_DATA) + pRAD->wMethodDataLength))
				{
					m_pCoreServerShell->ProcessNewClientMessage(
						m_pTransferClient,
						pRAD->nFromIP,
						pRAD->nFromRelayID,
						m_pGameStatus[i].nPlayerIndex, 
						(pData + sizeof(RELAY_ASKWAY_DATA) + pRAD->wMethodDataLength), 
						pRAD->routeDateLength);
				}
				else
				{
					
					m_pCoreServerShell->ProcessClientMessage(
						m_pGameStatus[i].nPlayerIndex, 
						(pData + sizeof(RELAY_ASKWAY_DATA) + pRAD->wMethodDataLength + 1), 
						pRAD->routeDateLength - 1);
				}
			}
			break;
		default:
			break;
		}
	}
}

void KSwordOnLineSever::TransferLoseWayMessageProcess(const char *pData, size_t dataLength)
{
	if (!m_pCoreServerShell || !m_pServer)
		return;

	EXTEND_HEADER* pInHeader = (EXTEND_HEADER*)pData;

	if (pInHeader->ProtocolFamily == pf_relay)
	{
		tagSearchWay *pSW = NULL;

		switch (pInHeader->ProtocolID)
		{
		case relay_c2c_data:
			pSW = (tagSearchWay *)(pData + sizeof(RELAY_DATA));
			break;
		case relay_c2c_askwaydata:
			pSW = (tagSearchWay *)(pData + sizeof(RELAY_ASKWAY_DATA) + ((RELAY_ASKWAY_DATA*)pData)->wMethodDataLength);
			break;
		default:
			return;
		}

		int lnID, nIndex,i;
		
		lnID = pSW->lnID;		//ÍøÂçID

		i =FindSame(lnID);

		//if (lnID < 0 || lnID >= m_nMaxGameStaus)
		//	return;
		if (!i)
		   return;

		if (m_pGameStatus[i].nExchangeStatus != enumExchangeSearchingWay)
			return;

		if (m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ID, pSW->nIndex, 0) != pSW->dwPlayerID)
			return;

		nIndex = m_pGameStatus[i].nPlayerIndex;

		tagNotifyPlayerExchange	npe;
		npe.cProtocol = s2c_notifyplayerexchange; //¿ç·þ ×ªµØÍ¼
		memset(&npe.guid, 0, sizeof(GUID));
		// if false, ip to 0
		npe.nIPAddr = 0;//TransDwIp;
		npe.nPort   = 0;//TransPort;
		m_pServer->SendData(lnID, &npe, sizeof(tagNotifyPlayerExchange));
//»Ö¸´¿ç·þÇ°×´Ì¬
		m_pCoreServerShell->RecoverPlayerExchange(nIndex);
//½âËø½ÇÉ«
		tagRoleEnterGame reg;
		reg.ProtocolType = c2s_roleserver_lock;
		reg.bLock = true;
		m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)reg.Name, nIndex);				
		if (m_pDatabaseClient)
			m_pDatabaseClient->SendPackToServer((const void *)&reg, sizeof(tagRoleEnterGame));

		m_pGameStatus[i].nExchangeStatus = enumExchangeBegin;
		m_pGameStatus[i].nGameStatus = enumPlayerPlaying;
	}
}

//GS´Ó¿Í·þ¶Ë»ñÈ¡°ï»áÏà¹ØÐÅÏ¢·¢ÍùS3°ï»á·þÎñÆ÷
void KSwordOnLineSever::ProcessPlayerTongMsg(const unsigned long nPlayerIdx, const char* pData, size_t dataLength)
{
	if (nPlayerIdx <= 0 || nPlayerIdx >= MAX_PLAYER)
		return;
	if (!pData)
		return;
	if (dataLength < sizeof(STONG_PROTOCOL_HEAD))
		return;
	int	nPLength = ((STONG_PROTOCOL_HEAD*)pData)->m_wLength;
	if (nPLength + 1 > dataLength)
		return;
	
	switch (((STONG_PROTOCOL_HEAD*)pData)->m_btMsgId)
	{
	// ·¢ÆðÉèÖÃ°ï»áÐÅÏ¢
	case enumTONG_COMMAND_ID_APPLY_SET_SEND:
	{
		  TONG_APPLY_SEND_SETINFO	*pSend = (TONG_APPLY_SEND_SETINFO*)pData;
		  STONG_SET_TONG_INFO_COMMAND setTong;
		  setTong.ProtocolFamily       = pf_alltong;
		  setTong.ProtocolID           = enumC2S_TONG_INFO_SET;
		  setTong.NextProtocolID	   = pSend->NextProtocolType;             //¾ßÌåµÄÉèÖÃÀàÐÍ
		  setTong.m_dwTongNameID       = pSend->m_dwTongNameID;        // 
		  setTong.m_dwNpcID            = pSend->m_NpcDwid;             //
		  setTong.m_dwParam            = pSend->m_inParam;
		  sprintf(setTong.m_title,pSend->m_Title);
		  //saTong.m_wLength = sizeof(STONG_GET_TONG_ATTACK_INFO_COMMAND);
		  if (m_pTongClient)
		    m_pTongClient->SendPackToServer((const void*)&setTong,sizeof(STONG_SET_TONG_INFO_COMMAND));
		  printf("---------·¢ËÍÉèÖÃ°ï»áÐÅÏ¢³É¹¦!----------- \n");
		  
	}
	break;
	case enumTONG_COMMAND_ID_APPLY_ATTACK_SEND:
		{   //·¢ÐûÕ½
			//¼ì²âÊÇ·ñÔÚÐûÕ½Ê±¼äÄÚ
			if (m_pCoreServerShell->GetGoldMissidVal(118)==0)
			{//µÈÓÚ0 ¾Í¼ì²â
				m_pCoreServerShell->SetAttackMsg(0,0,4);
				break;
			}
			//¿ªÊ¼·¢ËÍÐûÕ½ÐÅÏ¢
			TONG_APPLY_SEND_ATTACK	*pSend = (TONG_APPLY_SEND_ATTACK*)pData;
			STONG_GET_TONG_ATTACK_INFO_COMMAND	saTong;	
			//szPlayerName[0] = 0;
			//m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)szPlayerName, sAdd.m_nTargetIdx);//»ñÈ¡ÉêÇëÈËµÄÃû×Ö
			saTong.ProtocolFamily       = pf_alltong;
			saTong.ProtocolID           = enumC2S_TONG_ATTACK_SEND_INFO;  //
			saTong.m_dwTongNameID       = pSend->m_dwYTongNameID;         // 
			saTong.m_dwNpcID            = pSend->m_NpcDwid;               //·¢ËÍ·½µÄdwid ¹¥·½
			saTong.m_dwParam            = pSend->m_dwDTongNameID;;		  //Ä¿±ê·½µÄ°ï»á ÊØ·½
			//saTong.m_wLength = sizeof(STONG_GET_TONG_ATTACK_INFO_COMMAND);
			if (m_pTongClient)
				m_pTongClient->SendPackToServer((const void*)&saTong,sizeof(STONG_GET_TONG_ATTACK_INFO_COMMAND));
			printf("---------·¢ËÍ·¢Æð°ï»áÐûÕ½³É¹¦!----------- \n");

		}
		break;
	// »ñÈ¡ÐûÕ½Êý¾Ý
	case enumTONG_COMMAND_ID_APPLY_ATTACK_GET:
		{
		  TONG_APPLY_GET_ATTACK	*pApply = (TONG_APPLY_GET_ATTACK*)pData;
		  //char	szPlayerName[32];
		  STONG_GET_TONG_ATTACK_INFO_COMMAND	saTong;

		  //szPlayerName[0] = 0;
		  //m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)szPlayerName, sAdd.m_nTargetIdx);//»ñÈ¡ÉêÇëÈËµÄÃû×Ö
		  saTong.ProtocolFamily       = pf_alltong;
		  saTong.ProtocolID           = enumC2S_TONG_ATTACK_GET_INFO;  //
		  saTong.m_dwTongNameID       = pApply->m_dwTongNameID;        // 
		  saTong.m_dwNpcID            = pApply->m_NpcDwid;             //
		  saTong.m_dwParam            = 0;
		  //saTong.m_wLength = sizeof(STONG_GET_TONG_ATTACK_INFO_COMMAND);
		  //if (m_pTongClient)
	      //    m_pTongClient->SendPackToServer((const void*)&saTong,sizeof(STONG_GET_TONG_ATTACK_INFO_COMMAND));
		  //	printf("---------·¢ËÍ»ñÈ¡°ï»áÐûÕ½ÐÅÏ¢³É¹¦!-----------\n");
		}
		break;
	case enumTONG_COMMAND_ID_APPLY_CREATE:
		{//´´½¨°ï»á
			TONG_APPLY_CREATE_COMMAND	*pApply = (TONG_APPLY_CREATE_COMMAND*)pData;
			STONG_SERVER_TO_CORE_APPLY_CREATE	sApply;

			sApply.m_nCamp = pApply->m_btCamp;
			sApply.m_nPlayerIdx = nPlayerIdx;
			memcpy(sApply.m_szTongName, pApply->m_szName, sizeof(pApply->m_szName));

			// ´´½¨°ï»áÌõ¼þ¼ì²â
			int nRet = 0xff;
			if (m_pCoreServerShell)
				nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_APPLY_CREATE, (unsigned int)&sApply, 0);
			// ´´½¨°ï»áÌõ¼þ³ÉÁ¢
			if (nRet == 0)
			{
				char	szPlayerName[32];
				STONG_CREATE_COMMAND	sCreate;
               //»ñÈ¡´´½¨ÕßÃû×Ö
				m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)szPlayerName, nPlayerIdx);
				sCreate.ProtocolFamily = pf_tong;
				sCreate.ProtocolID = enumC2S_TONG_CREATE;
				sCreate.m_btCamp = pApply->m_btCamp;
				sCreate.m_dwParam = nPlayerIdx;
				sCreate.m_dwPlayerNameID = g_FileName2Id(szPlayerName);
				sCreate.m_btPlayerNameLength = strlen(szPlayerName);
				sCreate.m_btTongNameLength = strlen(sApply.m_szTongName);
				memcpy(sCreate.m_szBuffer,szPlayerName, sCreate.m_btPlayerNameLength);
				memcpy(&sCreate.m_szBuffer[sCreate.m_btPlayerNameLength], sApply.m_szTongName, sCreate.m_btTongNameLength);
				sCreate.m_wLength = sizeof(STONG_CREATE_COMMAND) - sizeof(sCreate.m_szBuffer) + sCreate.m_btTongNameLength + sCreate.m_btPlayerNameLength;
				if (m_pTongClient)//ÏòS3°ï»á·þÎñÆ÷·¢°ü
					m_pTongClient->SendPackToServer((const void*)&sCreate, sCreate.m_wLength);
				printf(">>>>>>>>>>(%s)·¢ËÍS3relay·þÎñÆ÷´´½¨°ï»á(%s)ÏûÏ¢£¬OK<<<<<<<<<<...\n",szPlayerName, sApply.m_szTongName);
				break;
			}
			// ´´½¨°ï»áÌõ¼þ²»³ÉÁ¢
			else
			{
				int		nNetID;
				TONG_CREATE_FAIL_SYNC	sFail;
				sFail.ProtocolType = s2c_extendtong;
				sFail.m_btMsgId = enumTONG_SYNC_ID_CREATE_FAIL;
				sFail.m_btFailId = nRet;
				sFail.m_wLength = sizeof(sFail) - 1;
				nNetID = m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NETID, nPlayerIdx, 0);
				if (m_pServer)
					m_pServer->PackDataToClient(nNetID, &sFail, sFail.m_wLength + 1);
                printf(">>>>>>>>>>(%d)·¢ËÍ¿Í»§¶Ë ´´½¨°ï»á(%s)Ê§°Ü£¬OK<<<<<<<<<<...\n",nPlayerIdx, sApply.m_szTongName);
				break;
			}
		}
		break;
	case enumTONG_COMMAND_ID_APPLY_ADD:
		{//ÉêÇë¼ÓÈë°ïÅÉ
			TONG_APPLY_ADD_COMMAND	*pApply = (TONG_APPLY_ADD_COMMAND*)pData;
			if (pApply->m_wLength != sizeof(TONG_APPLY_ADD_COMMAND) - 1)
				break;
			STONG_SERVER_TO_CORE_APPLY_ADD	sAdd;
			sAdd.m_nPlayerIdx = nPlayerIdx;
			sAdd.m_dwNpcID    = pApply->m_dwNpcID;
			if (m_pCoreServerShell)//»ñÈ¡¼ÓÈë°ï»áµÄÏûÏ¢
				m_pCoreServerShell->GetGameData(SGDI_TONG_APPLY_ADD, (unsigned int)&sAdd, 0);
		}
		break;
	case enumTONG_COMMAND_ID_ACCEPT_ADD:
		{//Í¬Òâ»ò¾Ü¾ø ¼ÓÈë°ï»á
			TONG_ACCEPT_MEMBER_COMMAND	*pAccept = (TONG_ACCEPT_MEMBER_COMMAND*)pData;
			if (pAccept->m_wLength != sizeof(TONG_ACCEPT_MEMBER_COMMAND) - 1)
				break;
			
			if (pAccept->m_nPlayerIdx  <= 0 || pAccept->m_nPlayerIdx  >= MAX_PLAYER) break;
			char	szPlayerName[32];
			szPlayerName[0] = 0;
			m_pCoreServerShell->GetGameData(SGDI_CHARACTER_NAME, (unsigned int)szPlayerName, pAccept->m_nPlayerIdx);//»ñÈ¡ÉêÇëÈËµÄÃû×Ö
			if (pAccept->m_btFlag == 0)
			{//°´Å¥µã»÷¾Ü¾ø
				STONG_SERVER_TO_CORE_REFUSE_ADD	sRefuse;
				sRefuse.m_nSelfIdx   = nPlayerIdx;             //±»ÉêÇëÈËµÄPlayerid
				sRefuse.m_nTargetIdx = pAccept->m_nPlayerIdx;  //ÉêÇëÈËµÄplayerid
				sRefuse.m_dwNameID   = g_FileName2Id(szPlayerName);//pAccept->m_dwNameID;    //ÉêÇëÈËµÄÃû×ÖID
				m_pCoreServerShell->OperationRequest(SSOI_TONG_REFUSE_ADD, (unsigned int)&sRefuse, 0);
				break;
			}
			else
			{//Í¬Òâ
				char	szTongName[32];
				STONG_SERVER_TO_CORE_CHECK_ADD_CONDITION sAdd;
				sAdd.m_nSelfIdx   = nPlayerIdx;              //±»ÉêÇëÈËµÄid
				sAdd.m_nTargetIdx = pAccept->m_nPlayerIdx;   //ÉêÇëÈËµÄPlayerid

				sAdd.m_dwNameID   = g_FileName2Id(szPlayerName);//pAccept->m_dwNameID;     //ÉêÇëÈËµÄÃû×ÖID

				int	nRet = FALSE;
				if (m_pCoreServerShell)//¼ì²âÌõ¼þ
					nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_CHECK_ADD_CONDITION, (unsigned int)szTongName, (unsigned int)&sAdd);
				// Ïò relay ÉêÇëÌí¼Ó°ïÖÚ
				if (nRet)
				{
					STONG_ADD_MEMBER_COMMAND	sTong;                        
					sTong.ProtocolFamily       = pf_tong;
					sTong.ProtocolID           = enumC2S_TONG_ADD_MEMBER;  //¼ÓÈë°ï»á
					sTong.m_dwParam            = sAdd.m_nTargetIdx;        //ÉêÇëÈËµÄPlayerid  
					sTong.m_dwPlayerNameID     = sAdd.m_dwNameID;          //ÉêÇëÈËµÄÃû×ÖID
					sTong.m_btPlayerNameLength = strlen(szPlayerName);     //ÉêÇëÈËµÄÃû×ÖID
					sTong.m_btTongNameLength   = strlen(szTongName);       //Òª¼ÓÈë°ï»áµÄÃû
					memcpy(sTong.m_szBuffer, szTongName, sTong.m_btTongNameLength);
					memcpy(&sTong.m_szBuffer[sTong.m_btTongNameLength], szPlayerName, sTong.m_btPlayerNameLength);
					sTong.m_wLength = sizeof(STONG_ADD_MEMBER_COMMAND) - sizeof(sTong.m_szBuffer) + sTong.m_btTongNameLength + sTong.m_btPlayerNameLength;
					if (m_pTongClient)
						m_pTongClient->SendPackToServer((const void*)&sTong, sTong.m_wLength);

				}
				else
				{//Ìõ¼þ²»³ÉÁ¢ ·¢ËÍ¾Ü¾øÐÅÏ¢
					STONG_SERVER_TO_CORE_REFUSE_ADD	sRefuse;
					sRefuse.m_nSelfIdx   = nPlayerIdx;             //±»ÉêÇëÈËµÄPlayerid
					sRefuse.m_nTargetIdx = pAccept->m_nPlayerIdx;  //ÉêÇëÈËµÄplayerid
					sRefuse.m_dwNameID   = g_FileName2Id(szPlayerName);;//pAccept->m_dwNameID;    //ÉêÇëÈËµÄÃû×ÖID
				    m_pCoreServerShell->OperationRequest(SSOI_TONG_REFUSE_ADD, (unsigned int)&sRefuse, 0);
					break;
				}
			}
		}
		break;
	case enumTONG_COMMAND_ID_APPLY_INFO:
		{//»ñÈ¡°ï»áÐÅÏ¢
			TONG_APPLY_INFO_COMMAND	*pInfo = (TONG_APPLY_INFO_COMMAND*)pData;
			if (pInfo->m_wLength < sizeof(TONG_APPLY_INFO_COMMAND) - 1 - sizeof(pInfo->m_szBuf))
				break;
			STONG_SERVER_TO_CORE_GET_INFO	sGet;

			switch (pInfo->m_btInfoID)
			{
			case enumTONG_APPLY_INFO_ID_SELF: //×Ô¼º²éÑ¯×Ô¼º°ï»áµÄÐÅÏ¢
				sGet.m_nSelfIdx = nPlayerIdx;
				sGet.m_nInfoID = pInfo->m_btInfoID;
				sGet.m_nParam1 = pInfo->m_nParam1;
				sGet.m_nParam2 = pInfo->m_nParam2;
				sGet.m_nParam3 = pInfo->m_nParam3;
				sGet.m_szName[0] = 0;
				if (m_pCoreServerShell)
					m_pCoreServerShell->GetGameData(SGDI_TONG_GET_INFO, (unsigned int)&sGet, 0);
				break;
			case enumTONG_APPLY_INFO_ID_MASTER:
				break;//°ïÖ÷
			case enumTONG_APPLY_INFO_ID_DIRECTOR:
				break;//³¤ÀÏ
			case enumTONG_APPLY_INFO_ID_MANAGER:
				{//¶Ó³¤
					int nRet = 0;
					sGet.m_nSelfIdx = nPlayerIdx;
					sGet.m_nInfoID = pInfo->m_btInfoID;
					sGet.m_nParam1 = pInfo->m_nParam1;
					sGet.m_nParam2 = pInfo->m_nParam2;
					sGet.m_nParam3 = pInfo->m_nParam3;
					sGet.m_szName[0] = 0;
					if (m_pCoreServerShell)
						nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_GET_INFO, (unsigned int)&sGet, 0);
					if (nRet == 0)
						break;

					STONG_GET_MANAGER_INFO_COMMAND	sGet;
					sGet.ProtocolFamily = pf_tong;
					sGet.ProtocolID = enumC2S_TONG_GET_MANAGER_INFO;
					sGet.m_dwParam = nPlayerIdx;
					sGet.m_nParam1 = pInfo->m_nParam1;
					sGet.m_nParam2 = pInfo->m_nParam2;
					sGet.m_nParam3 = pInfo->m_nParam3;
					if (m_pTongClient)//S3
						m_pTongClient->SendPackToServer((const void*)&sGet, sizeof(sGet));
				}
				break;
			case enumTONG_APPLY_INFO_ID_MEMBER:
				{//°ïÖÚ
					int nRet = 0;
					sGet.m_nSelfIdx = nPlayerIdx;
					sGet.m_nInfoID = pInfo->m_btInfoID;
					sGet.m_nParam1 = pInfo->m_nParam1;	  //tongid
					sGet.m_nParam2 = pInfo->m_nParam2;	  //»ñÈ¡Êý¾ÝµÄ¿ªÊ¼Î»ÖÃ
					sGet.m_nParam3 = pInfo->m_nParam3;	  //Ã¿´Î»ñÈ¡µÄÊýÁ¿
					sGet.m_szName[0] = 0;
					if (m_pCoreServerShell)
						nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_GET_INFO, (unsigned int)&sGet, 0);
					if (nRet == 0)
						break;

					STONG_GET_MEMBER_INFO_COMMAND	sGet;
					sGet.ProtocolFamily = pf_tong;
					sGet.ProtocolID = enumC2S_TONG_GET_MEMBER_INFO;
					sGet.m_dwParam = nPlayerIdx;
					sGet.m_nParam1 = pInfo->m_nParam1;
					sGet.m_nParam2 = pInfo->m_nParam2;
					sGet.m_nParam3 = pInfo->m_nParam3;
					if (m_pTongClient)//S3
						m_pTongClient->SendPackToServer((const void*)&sGet, sizeof(sGet));
				}
				break;
			case enumTONG_APPLY_INFO_ID_LIST:
				{//»ñÈ¡ËùÓÐ°ï»áÁÐ±í
					/*int nRet = 0;
					sGet.m_nSelfIdx = nPlayerIdx;
					sGet.m_nInfoID = pInfo->m_btInfoID;
					sGet.m_nParam1 = pInfo->m_nParam1;	  //tongid
					sGet.m_nParam2 = pInfo->m_nParam2;	  //»ñÈ¡Êý¾ÝµÄ¿ªÊ¼Î»ÖÃ
					sGet.m_nParam3 = pInfo->m_nParam3;	  //Ã¿´Î»ñÈ¡µÄÊýÁ¿
					sGet.m_szName[0] = 0;
					if (m_pCoreServerShell)
						nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_GET_INFO, (unsigned int)&sGet, 0);
					if (nRet == 0)
						break;*/
					STONG_GET_LIST_INFO_COMMAND	sGet;
					sGet.ProtocolFamily = pf_alltong;
					sGet.ProtocolID = enumC2S_TONG_GET_LIST_INFO;
					sGet.m_dwParam = nPlayerIdx;
					sGet.m_nParam1 = pInfo->m_nParam1;	  //tongid
					sGet.m_nParam2 = pInfo->m_nParam2;	  //»ñÈ¡Êý¾ÝµÄ¿ªÊ¼Î»ÖÃ
					sGet.m_nParam3 = pInfo->m_nParam3;	  //Ã¿´Î»ñÈ¡µÄÊýÁ¿
					if (m_pTongClient)//S3
						m_pTongClient->SendPackToServer((const void*)&sGet, sizeof(sGet));
					//printf("---------·¢ËÍ°ï»áÁÐ±í³É¹¦!-----------\n");
				}
				break;
			case enumTONG_APPLY_INFO_ID_ONE:
				break;
			case enumTONG_APPLY_INFO_ID_TONG_HEAD:
				{//¿Í»§¶Ë´«¹ýÀ´µÄÊý¾Ý
					DWORD	dwTongNameID = 0;

					sGet.m_nSelfIdx = nPlayerIdx;
					sGet.m_nInfoID = pInfo->m_btInfoID;
					sGet.m_nParam1 = pInfo->m_nParam1;
					sGet.m_nParam2 = pInfo->m_nParam2;
					sGet.m_nParam3 = pInfo->m_nParam3;
					sGet.m_szName[0] = 0;

					if (m_pCoreServerShell)
						dwTongNameID = m_pCoreServerShell->GetGameData(SGDI_TONG_GET_INFO, (unsigned int)&sGet, 0);
					if (dwTongNameID == 0)
						break;
                    //½«¿Í»§¶ËµÄÇëÇó·¢ÍùS3°ï»á·þÎñÆ÷ ---»ñÈ¡°ï»á°ïÖ÷ºÍ³¤ÀÏµÄÐÅÏ¢
					STONG_GET_TONG_HEAD_INFO_COMMAND	sGet;
					sGet.ProtocolFamily	= pf_tong;
					sGet.ProtocolID		= enumC2S_TONG_GET_HEAD_INFO;
					sGet.m_dwParam		= nPlayerIdx;
					sGet.m_dwNpcID		= pInfo->m_nParam1;
					sGet.m_dwTongNameID	= dwTongNameID;
					if (m_pTongClient)//S3
						m_pTongClient->SendPackToServer((const void*)&sGet, sizeof(sGet));
				}
				break;
			}
		}
		break;
	case enumTONG_COMMAND_ID_APPLY_INSTATE:
		{//ÈÎÃâÖ°Î»
			TONG_APPLY_INSTATE_COMMAND	*pApply = (TONG_APPLY_INSTATE_COMMAND*)pData;
			if (pApply->m_wLength + 1 != sizeof(TONG_APPLY_INSTATE_COMMAND))
				break;
			int nRet = 0;
			if (m_pCoreServerShell)
				nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_INSTATE_POWER, (unsigned int)pApply, nPlayerIdx);
			if (nRet == 0)
				break;

			STONG_INSTATE_COMMAND	sInstate;
			sInstate.ProtocolFamily	= pf_tong;
			sInstate.ProtocolID		= enumC2S_TONG_INSTATE;
			sInstate.m_btCurFigure	= pApply->m_btCurFigure;
			sInstate.m_btCurPos		= pApply->m_btCurPos;
			sInstate.m_btNewFigure	= pApply->m_btNewFigure;
			sInstate.m_btNewPos		= pApply->m_btNewPos;
			sInstate.m_dwParam		= nPlayerIdx;
			sInstate.m_dwTongNameID	= pApply->m_dwTongNameID;
			memset(sInstate.m_szName, 0, sizeof(sInstate.m_szName));
			memcpy(sInstate.m_szName, pApply->m_szName, pApply->m_wLength + 1 + sizeof(pApply->m_szName) - sizeof(TONG_APPLY_INSTATE_COMMAND));

			if (m_pTongClient)
				m_pTongClient->SendPackToServer((const void*)&sInstate, sizeof(sInstate));
		}
		break;

	case enumTONG_COMMAND_ID_APPLY_KICK:
		{
			TONG_APPLY_KICK_COMMAND	*pKick = (TONG_APPLY_KICK_COMMAND*)pData;
			if (pKick->m_wLength + 1 != sizeof(TONG_APPLY_KICK_COMMAND))
				break;
			int nRet = 0;
			if (m_pCoreServerShell)
				nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_KICK_POWER, (unsigned int)pKick, nPlayerIdx);
			if (nRet == 0)
				break;

			STONG_KICK_COMMAND	sKick;
			sKick.ProtocolFamily	= pf_tong;
			sKick.ProtocolID		= enumC2S_TONG_KICK;
			sKick.m_dwParam			= nPlayerIdx;
			sKick.m_dwTongNameID	= pKick->m_dwTongNameID;
			sKick.m_btFigure		= pKick->m_btFigure;
			sKick.m_btPos			= pKick->m_btPos;
			memcpy(sKick.m_szName, pKick->m_szName, sizeof(pKick->m_szName));

			if (m_pTongClient)
				m_pTongClient->SendPackToServer((const void*)&sKick, sizeof(sKick));
		}
		break;
	case enumTONG_COMMAND_ID_APPLY_LEAVE:
		{
			TONG_APPLY_LEAVE_COMMAND	*pLeave = (TONG_APPLY_LEAVE_COMMAND*)pData;
			if (pLeave->m_wLength + 1 != sizeof(TONG_APPLY_LEAVE_COMMAND))
				break;
			int nRet = 0;
			if (m_pCoreServerShell)
				nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_LEAVE_POWER, (unsigned int)pLeave, nPlayerIdx);
			if (nRet == 0)
				break;

			STONG_LEAVE_COMMAND	sLeave;
			sLeave.ProtocolFamily	= pf_tong;
			sLeave.ProtocolID		= enumC2S_TONG_LEAVE;
			sLeave.m_dwParam		= nPlayerIdx;
			sLeave.m_dwTongNameID	= pLeave->m_dwTongNameID;
			sLeave.m_btFigure		= pLeave->m_btFigure;
			sLeave.m_btPos			= pLeave->m_btPos;
			memcpy(sLeave.m_szName, pLeave->m_szName, sizeof(pLeave->m_szName));

			if (m_pTongClient)
				m_pTongClient->SendPackToServer((const void*)&sLeave, sizeof(sLeave));
		}
		break;
	case enumTONG_COMMAND_ID_APPLY_CHANGE_MASTER:
		{//´«Î»
			TONG_APPLY_CHANGE_MASTER_COMMAND	*pChange = (TONG_APPLY_CHANGE_MASTER_COMMAND*)pData;
			if (pChange->m_wLength + 1 != sizeof(TONG_APPLY_CHANGE_MASTER_COMMAND))
				break;
			int nRet = 0;
			if (m_pCoreServerShell)
				nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_CHANGE_MASTER_POWER, (unsigned int)pChange, nPlayerIdx);
			if (nRet == 0)
				break;

			STONG_CHANGE_MASTER_COMMAND	sChange;
			sChange.ProtocolFamily	= pf_tong;
			sChange.ProtocolID		= enumC2S_TONG_CHANGE_MASTER;
			sChange.m_btFigure		= pChange->m_btFigure;
			sChange.m_btPos			= pChange->m_btPos;
			sChange.m_dwParam		= nPlayerIdx;
			sChange.m_dwTongNameID	= pChange->m_dwTongNameID;
			memcpy(sChange.m_szName, pChange->m_szName, sizeof(pChange->m_szName));
			if (m_pTongClient)
				m_pTongClient->SendPackToServer((const void*)&sChange, sizeof(sChange));
		}
		break;
	case enumTONG_COMMAND_ID_APPLY_SAVE:
	case enumTONG_COMMAND_ID_APPLY_GET:
	case enumTONG_COMMAND_ID_APPLY_SND:
			TONG_APPLY_SAVE_COMMAND	*pChange = (TONG_APPLY_SAVE_COMMAND*)pData;
			if (pChange->m_wLength + 1 != sizeof(TONG_APPLY_SAVE_COMMAND))
				break;
			int nRet = 0;
			if (m_pCoreServerShell)
				nRet = m_pCoreServerShell->GetGameData(SGDI_TONG_MONEY_POWER, (unsigned int)pChange, nPlayerIdx);
			if (nRet == 0)
				break;

			STONG_MONEY_COMMAND	sChange;
			sChange.ProtocolFamily	= pf_tong;
			if (pChange->m_btMsgId == enumTONG_COMMAND_ID_APPLY_SAVE)
				sChange.ProtocolID		= enumC2S_TONG_MONEY_SAVE;
			else if (pChange->m_btMsgId == enumTONG_COMMAND_ID_APPLY_GET)
				sChange.ProtocolID		= enumC2S_TONG_MONEY_GET;
			else sChange.ProtocolID		= enumC2S_TONG_MONEY_SND;
			sChange.m_dwMoney		= pChange->m_dwMoney;
			sChange.m_dwParam		= nPlayerIdx;
			sChange.m_dwTongNameID	= pChange->m_dwTongNameID;
			memcpy(sChange.m_szName, pChange->m_szName, sizeof(pChange->m_szName));
			if (m_pTongClient)
					m_pTongClient->SendPackToServer((const void*)&sChange, sizeof(sChange));
		break;
	}
}

BOOL KSwordOnLineSever::CheckPlayerID(unsigned long netidx, DWORD nameid)
{
	//if (netidx < 0 || netidx >= m_nMaxGameStaus)
	//	return FALSE;

	int i =FindSame(netidx);

	if (!i)
		return FALSE;

	int idx = m_pGameStatus[i].nPlayerIndex;
	if (idx <= 0 || idx >= MAX_PLAYER)
		return FALSE;

	if (nameid != m_pCoreServerShell->GetGameData(SGDI_CHARACTER_ID, idx, 0))
		return FALSE;

	return TRUE;
}


////////////////////////////////////////////
//»ñÈ¡»úÆ÷Âë 
/* int  KSwordOnLineSever::getMAC(char *mac)
{ 
  unsigned long s1,s2;
  unsigned char vendor_id[]="--------";//CPUÌá¹©ÉÌID
  char *cpu32;
  char *cpu64;
  char *cpuscs;
   _asm    // µÃµ½CPUÌá¹©ÉÌÐÅÏ¢ 
   {   
	      xor eax,eax   // ½«eaxÇå0
		  cpuid         // »ñÈ¡CPUIDµÄÖ¸Áî
		  mov dword ptr vendor_id,ebx
		  mov dword ptr vendor_id[+4],edx
		  mov dword ptr vendor_id[+8],ecx  
   } 
   sprintf(cpuscs,"%s",vendor_id);

  _asm    // µÃµ½CPU IDµÄ¸ß32Î» 
  { 
	  mov eax,01h    
      xor edx,edx
      cpuid
      mov s2,eax
  }
  sprintf(cpu32,"%08X",s2);

  _asm    // µÃµ½CPU IDµÄµÍ64Î»
  { 
	  mov eax,03h
      xor ecx,ecx
      xor edx,edx
      cpuid
      mov s1,edx
      mov s2,ecx
  }
  sprintf(cpu64,"%04X%04X",s1,s2);

	NCB ncb;           
	typedef struct _ASTAT_           
	{ 
		ADAPTER_STATUS       adapt;     
		NAME_BUFFER   NameBuff[30];           
	}ASTAT, *PASTAT; 
	
	ASTAT Adapter;           
	typedef struct _LANA_ENUM           
	{ 
		UCHAR   length;     
		UCHAR   lana[MAX_LANA];           
	}LANA_ENUM;           
	
	LANA_ENUM lana_enum;                 
	UCHAR   uRetCode;           
	memset(&ncb,   0,   sizeof(ncb));           
	memset(&lana_enum,   0,   sizeof(lana_enum));           
	
	ncb.ncb_command  =   NCBENUM;           
	ncb.ncb_buffer   =   (unsigned   char   *)&lana_enum;           
	ncb.ncb_length   =   sizeof(LANA_ENUM);           
	uRetCode         =   Netbios(&ncb);           
	if(uRetCode != NRC_GOODRET)           
		return  uRetCode;           
	
	for(int lana=0; lana <lana_enum.length; ++lana)           
	{ 
		ncb.ncb_command  =   NCBRESET;   
		ncb.ncb_lana_num =   lana_enum.lana[lana];   
		uRetCode         =   Netbios(&ncb);       
		if(uRetCode ==NRC_GOODRET)     
			break;   
	}   
	if(uRetCode!=NRC_GOODRET) 
		return uRetCode;           	
	memset(&ncb,0, sizeof(ncb));     
	ncb.ncb_command    =   NCBASTAT;     
	ncb.ncb_lana_num   =   lana_enum.lana[0]; 
	strcpy((char*)ncb.ncb_callname,   "* ");   
	ncb.ncb_buffer     =   (unsigned   char   *)&Adapter; 
	ncb.ncb_length     =   sizeof(Adapter); 
	uRetCode           =   Netbios(&ncb);     
	if(uRetCode != NRC_GOODRET)       
		return   uRetCode;           
	sprintf(mac, "%X%02X%02X%02X%02X%02X%08X%08X",           
		Adapter.adapt.adapter_address[0],           
		Adapter.adapt.adapter_address[1],           
		Adapter.adapt.adapter_address[2],           
		Adapter.adapt.adapter_address[3],           
		Adapter.adapt.adapter_address[4],           
		Adapter.adapt.adapter_address[5],
		cpu32,
		cpu64
		); 
	return   0;       
} 
 */

void KSwordOnLineSever::EDPad_Encipher(char* pPlaintext, int nPlainLen)
 {
	 if (pPlaintext && nPlainLen > 0)
	 {	
		 unsigned char	*pContent = (unsigned char*)pPlaintext;
#define	NUM_FIX_CIPHER_CHARA	3
		 unsigned char cCipher[NUM_FIX_CIPHER_CHARA + NUM_FIX_CIPHER_CHARA] = { 'W', 'o', 'o', 'y', 'u', 'e'};
		 if (nPlainLen > NUM_FIX_CIPHER_CHARA)
		 {
			 cCipher[0] = pContent[nPlainLen - 1] % 4;
			 if (cCipher[0])
				 cCipher[0] = cCipher[cCipher[0] + 2];
			 else
				 cCipher[0] = pContent[nPlainLen - 1];
			 
			 if (nPlainLen > NUM_FIX_CIPHER_CHARA + 1)
			 {
				 cCipher[1] = 'l';
				 if (nPlainLen > NUM_FIX_CIPHER_CHARA + 2)
				 {
					 cCipher[2] = pContent[nPlainLen - 2];
					 if (nPlainLen > NUM_FIX_CIPHER_CHARA + 3)
						 cCipher[1] =pContent[nPlainLen - 3];
				 }
			 }
		 }
		 for (int nPos = 0; nPos < nPlainLen; nPos++)
		 {
			 pContent[nPos] = (pContent[nPos] - 0x20 + (0xff - cCipher[cCipher[0] % 3])) % 0xdf + 0x20;
			 cCipher[0] = cCipher[1];	cCipher[1] = cCipher[2];
			 cCipher[2] = pContent[nPos];
		 }
	 }
 }


bool KSwordOnLineSever::JieMa_Decipher(char* pInTxtPass,char *pOutTxtPass)
{

	char  szfdfd[64]={0};
	miracl *mip=mirsys(600,16);//³õÊ¼»¯600Î»µÄ10½øÐÐÖÆÊý
	mip->IOBASE=16;	//16½øÖÆÄ£Ê½
	//¶¨Òå²¢³õÊ¼»¯±äÁ¿
	big m=mirvar(0);	//m ·ÅÃ÷ÎÄ£º×¢²áÂëSN
	big c=mirvar(0);	//c ·ÅÃÜÎÄ£ºÓÃ»§ÃûName
	big n=mirvar(0);	//n Ä£Êý
	big e=mirvar(0);	//e ¹«Ô¿

	TCHAR SN[256]={0};
	TCHAR temp[256]={0};
	//TCHAR strMacAddr[256]={0};
	sprintf(SN,pInTxtPass);
	//¼ì²éSNÊÇ·ñÎª16½øÖÆ
	int	len =strlen(SN),i,j;
	for (i=0,j=0;i<len;++i)
	{
		if(isxdigit(SN[i])==0)
		{
			j=1;
			break;
		}
	} 	
	//Èç¹ûÊäÈëµÄSNÎª16½øÖÆÇÒ³¤¶È²»Îª0
	if (j!=1)
	{	
		cinstr(m,SN);	  								//×¢²áÂë»»³ÉBIG
		cinstr(n,"A7404030B471A60D33917658A42664ED");	//³õÊ¼»¯Ä£Êýn	 
		cinstr(e,"19421");								//³õÊ¼»¯¹«Ô¿e  ÖÊÊý
		//µ±m<nÊ±
		if(mr_compare(m,n)==-1)
		{
			powmod(m,e,n,c);//¼ÆËãc=m^e mod n  //½âÃÜ
			cotstr(c,temp);
			//ÊÍ·ÅÄÚ´æ
			mirkill(m);
			mirkill(c);
			mirkill(n);
			mirkill(e);
			mirexit();
		}
		else
			j=1;		
	}

	if(j==1)
	{//½âÃÜÊ§°Ü  
		return false;
	}
	else
	{//½âÃÜ³É¹¦ 
		strcpy(pOutTxtPass,temp);
		return true;
	}
}
 
 //--------------------------------------------------------------------------
 //	¹¦ÄÜ£ºÒ»´ÎÒ»ÃÜÂëÂÒ±¾½âÂëº¯Êý
 //--------------------------------------------------------------------------
void KSwordOnLineSever::EDPad_Decipher(char* pCiphertext, int nCiphertextLen)
 {
	 if (pCiphertext && nCiphertextLen > 0)
	 {
		 unsigned char* pContent = (unsigned char*)pCiphertext;
#define	NUM_FIX_CIPHER_CHARA	3
		 unsigned char cCipher[NUM_FIX_CIPHER_CHARA + NUM_FIX_CIPHER_CHARA] = { 'W', 'o', 'o', 'y', 'u', 'e'};
		 int nPos;
		 for (nPos = nCiphertextLen - 1; nPos >= NUM_FIX_CIPHER_CHARA; nPos--)
			 pContent[nPos] = (pContent[nPos] - 0x20 + (pContent[nPos - 3 + (pContent[nPos - 3] % 3)] - 0x20)) % 0xdf + 0x20;
		 if (nCiphertextLen > NUM_FIX_CIPHER_CHARA)
		 {
			 cCipher[0] = pContent[nCiphertextLen - 1] % 4;
			 if (cCipher[0])
				 cCipher[0] = cCipher[cCipher[0] + 2];
			 else
				 cCipher[0] = pContent[nCiphertextLen - 1];
			 
			 if (nCiphertextLen > NUM_FIX_CIPHER_CHARA + 1)
			 {
				 cCipher[1] = 'l';
				 if (nCiphertextLen > NUM_FIX_CIPHER_CHARA + 2)
				 {
					 cCipher[2] = pContent[nCiphertextLen - 2];
					 if (nCiphertextLen > NUM_FIX_CIPHER_CHARA + 3)
						 cCipher[1] =pContent[nCiphertextLen - 3];
				 }
			 }
		 }
		 
		 if (nPos)
			 memcpy(cCipher + NUM_FIX_CIPHER_CHARA, pContent, nPos);
		 
		 for (;nPos >= 0; nPos--)
		 {
			 pContent[nPos] = (pContent[nPos] - 0x20 + (cCipher[nPos + (cCipher[nPos] % 3)] - 0x20)) % 0xdf + 0x20;
		 }
	 }
 }

VOID  CALLBACK  KSwordOnLineSever::TimerProc(HWND hWnd,UINT nMsg,UINT nTimerid,DWORD dwTime)   
{   
	switch(nTimerid)   
	{   
	case   1:   ///´¦ÀíIDÎª1µÄ¶¨Ê±Æ÷   
		//Do1(); 
		 KillTimer(GetWindowHwnd(),1);
		 printf("¹Ø±Õ¼ÆËãÆ÷³É¹¦£¡");
		break;   
	case   2:   ///´¦ÀíIDÎª2µÄ¶¨Ê±Æ÷   
		//Do2();   
		break;   
	}   
}   
