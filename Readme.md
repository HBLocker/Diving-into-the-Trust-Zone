The trust zone is a hardware method used to improve software security. It is a way to isolate critical components in a system by hardware separation. Calls are made from non-secure world to secure word calling Trusted Applications (TA’s) 

Today the Trustzone is a somewhat intimidating stack which is used in many devices, it is mainly recognised to be used within Smartphones, but it is also used within the automotive industry. The TrustZone was first used by Nokia in 2004, this was added into the Nokia N8900.  

A common Trusted Execution environment used is OP-TEE Opensource Trusted Execution Environment (OP-TEE) this was developed back in 2013 and is commonly used today for Trusted Applications and the TrustZone. 

```


   User space                  Kernel                   Secure world
   ~~~~~~~~~~                  ~~~~~~                   ~~~~~~~~~~~~
+--------+                                             +-------------+
| Client |                                             | Trusted     |
+--------+                                             | Application |
   /\                                                  +-------------+
   || +----------+                                           /\
   || |tee-      |                                           ||
   || |supplicant|                                           \/
   || +----------+                                     +-------------+
   \/      /\                                          | TEE Internal|
+-------+  ||                                          | API         |
+ TEE   |  ||            +--------+--------+           +-------------+
| Client|  ||            | TEE    | OP-TEE |           | OP-TEE      |
| API   |  \/            | subsys | driver |           | Trusted OS  |
+-------+----------------+----+-------+----+-----------+-------------+
|      Generic TEE API        |       |     OP-TEE MSG               |
|      IOCTL (TEE_IOC_*)      |       |     SMCCC (OPTEE_SMC_CALL_*) |
+-----------------------------+       +------------------------------+

```

Commands can be used to communicate between the non-secure world and the secure world. The TEE supports a client api which is used to communicate with the TEE application in secure world, there are many commands which are used, these follow a standard for all trusted related applications, in the TPM for example the commands are identical, just the identifier is different and TSS_CommandName. The purpose of these are to talk from Non secure world to secure world. 



```
                                                                                
        ####################                                  ######################                                
        #Client Application#                 #                #Trusteed Application#         
        ####################                                  ######################                     
       TEEC_Context               <---------------------->		TA_CreateEntryPoint     
            .   			     #                                .
       TEEC_Opensession           <---------------------->		TA_OpenSessionEntryPoint 
            .                                #                                .  
       TEEC_InvokeCommand         <---------------------->		TA_InvokeCommandEntryPoint 
            .             	             #                                .  
       TEEC_CloseSession	  <---------------------->		TA_CloseSessionEntryPoint 
            .                     	     #                                .  
       TEEC_FinalizeContext       <---------------------->		TA_DestoryEntryPoint 

```

  ### TEEC_Context               <---------------------->	         	TA_CreateEntryPoint   
 This function initializes a new TEE Context, forming a connection between this Client Application and the
TEE identified by the string identifier name. 

```c
 
 TEEC_InitializeContext() - Initializes a context holding connection
 * information on the specific TEE, designated by the name string.

TEEC_Result TEEC_InitializeContext(const char *name, TEEC_Context *context); 

```
TEEC_Context is defined below as so:
```
/**
 * struct TEEC_Context - Represents a connection between a client application
 * and a TEE.
 */
typedef struct {
	/* Implementation defined */
	int fd;
	bool reg_mem;
	bool memref_null;
} TEEC_Context;
```

How does this communicate with TA_CreateEntryPoint?
```c
TEE_Result TA_EXPORT TA_CreateEntryPoint(void);
```
 
This is the applications cotructor. Which will create a new instance of the TA. If the application is created sucsessfully it will return the following:
```c
- TEE_SUCCESS:
````

After TA_SUCSESS has been invoked successfully  TA_InvokeCommandEntryPoint will be called, it takes in the following params and the context from TEEC_InitializeContext.
```c
TEE_Result TA_EXPORT TA_InvokeCommandEntryPoint(void *sessionContext,
				uint32_t commandID,
				uint32_t paramTypes,
				TEE_Param params[TEE_NUM_PARAMS]);
        
- sessionContext: The value of the void* opaque data pointer set by the
 *   Trusted Application in the function TA_OpenSessionEntryPoint.
 * - commandID: A Trusted Application-specific code that identifies the
 *   command to be invoked.
 * - paramTypes: the types of the four parameters.
 * - params: a pointer to an array of four parameters.                
```
###  TEEC_Opensession           <---------------------->		TA_OpenSessionEntryPoint 

This function opens a new Session between the Client Application and the specified Trusted Application


Next  OpenSession is invoked which follows the following Structure:
```c
TEEC_Result TEEC_OpenSession(TEEC_Context *context,
			     TEEC_Session *session,
			     const TEEC_UUID *destination,
			     uint32_t connectionMethod,
			     const void *connectionData,
			     TEEC_Operation *operation,
			     uint32_t *returnOrigin);
           
* @param context            The initialized TEE context structure in which
 *                           scope to open the session.
 * @param session            The session to initialize.
 * @param destination        A structure identifying the trusted application
 *                           with which to open a session.
 *
 * @param connectionMethod   The connection method to use.
 * @param connectionData     Any data necessary to connect with the chosen
 *                           connection method. Not supported, should be set to
 *                           NULL.
 * @param operation          An operation structure to use in the session. May
 *                           be set to NULL to signify no operation structure
 *                           needed.
 *
 * @param returnOrigin       A parameter which will hold the error origin if
 *                           this function returns any value other than
 *                           TEEC_SUCCESS.
 *
```
In the Trusted Space OpensessionentryPoint is invoked:

```c
TEE_Result TA_EXPORT TA_OpenSessionEntryPoint(uint32_t paramTypes,
				TEE_Param params[TEE_NUM_PARAMS],
				void **sessionContext);
     
 * Parameters:
 * - paramTypes: the types of the four parameters.
 * - params: a pointer to an array of four parameters.
 * - sessionContext: A pointer to a variable that can be filled by the
 *   Trusted Application instance with an opaque void* data pointer

```
### TEEC_CloseSession	  <---------------------->		TA_CloseSessionEntryPoint 
 This function closes a Session which has been opened with a Trusted Application.
All Commands within the Session MUST have completed before calling this function.
The Implementation MUST do nothing if the session parameter is NULL                                                                               
After the session has been created the session is then closed with TEEC_CloseSession:

```c
void TEEC_CloseSession(TEEC_Session *session);

* TEEC_CloseSession() - Closes the session which has been opened with the
 * specific trusted application.

```
Then the trusted application is then closed:
```
void TA_EXPORT TA_CloseSessionEntryPoint(void *sessionContext);
```

Finaly the context is destroyed meaning it can not be ran and will have to be reinitialized again with TEEC_Context.
```c
void TEEC_FinalizeContext(TEEC_Context *context);

Destroys a context holding connection information on the specific TEE.
```

### Trusted Kernel -  Gatekeeper 

Now we have some understadning of the TrustZone, we should dive into it and see how this works....

I was going to use QTEE but I feel I would not be writing anything of use. I deviced to look into how Gatekeeping works and the use in mobile phones. After some Googling, I found someone had dumped the contents of their Blackview BV6100 online. The libary we are going to be looking into is gatekeeper.trustkernel.so 

![image](https://www.doer.ch/media/catalog/product/cache/1/image/470x470/9df78eab33525d08d6e5fb8d27136e95/m/p/mp7049-1_1.jpg)





### What is a GateKeeper and HAL?

The main purpose of the Gatekeeper function is used to verify the password lock pattern of the device. When the user verifies their passwords stored on an android device, the Gatekeeper uses the TEE-shared secret to sign an authentication to the hardware-backed keystore. 

The hardware abstraction layer (HAL) is part of the Android OS and runs within user space. 

This shields the implementation details of the hardware driver module downward, providing hardware access service upward. Through the Hardware Abstraction Layer (HAL)
The Android system is divided into two main layers, supporting hardware devices on one layer in user space and the other layer in Kernel space. 

The HAL of Android manages many hardware access interfaces from modules, each module had a dynamic link library eg foo.so  These libraries need to obey the android specification, otherwise they will not work. In Android each hardware abstraction layer module is described by hw_module_t and each device by hw_device_t

![image](https://source.android.com/security/images/gatekeeper-flow.png)

### Initialization 

The first stage is to initialize the device, first the device is opened, which returns the ID,  this means it needs to know what the ID of the device. TrustKernelGateKeeperDevice is used to get hw_module_t which is a uint_32 tag, if the device is the correct, the authentication method will begin. 

```c++
gatekeeper::TrustKernelGateKeeperDevice::TrustKernelGateKeeperDevice
(TrustKernelGateKeeperDevice *this,hw_module_t *param_1)
{
  __android_log_print(4,"GatekeeperHAL","Init device\n");
 
  *(undefined8 *)(this + 0x18) = 0;
  *(undefined8 *)(this + 0x10) = 0;
  *(hw_module_t **)(this + 8) = param_1;
  *(undefined8 *)this = 0x148574454;
  
  //important stuff 
  *(code **)(this + 0x70) = close_device;
  *(code **)(this + 0x78) = enroll;
  *(code **)(this + 0x80) = verify;
  *(undefined8 *)(this + 0x88) = 0;
  OpenTEESession(this);
  *(undefined4 *)(this + 0x98) = 0;
  return;
}
```


### Close_Device 

close_device is used to close the device, TEEC_CloseSession(param_1 + 0x1a8); is first called, closing the session created. Then TEEC_FinalizeContext is then called which then closed the connection. Finaly operator.delete(param_1); is called which deeltes hw_delete_t. 

```c++
undefined8 gatekeeper::TrustKernelGateKeeperDevice::close_device(hw_device_t *param_1)

{
  if (param_1 != (hw_device_t *)0x0) {
    __android_log_print(4,"GatekeeperHAL","Close device\n");
    TEEC_CloseSession(param_1 + 0x1a8);
    TEEC_FinalizeContext(param_1 + 0xa0);
    operator.delete(param_1);
  }
  return 0;
}



```
### enroll



### verify 
The verify function is used to verifiy the devices. The params it takes in are the lengh of the password, from the Android documentation we can put the two together to get an underdtanding of what is happening here:

```c++
//Based from Androids documentation 
undefined8 gatekeeper::TrustKernelGateKeeperDevice::verify(const struct gatekeeper_device *dev, uid,  challenge, *enrolled_password_handle,  enrolled_password_handle_length, *provided_password,provided_password_length, **auth_token, *auth_token_length, *request_reenroll)
```

```c++
//gatekeeper.trustedkernel.so
undefined8 gatekeeper::TrustKernelGateKeeperDevice::verify
          (gatekeeper_device *param_1,uint param_2,ulong param_3,uchar *param_4,uint param_5,
          uchar *param_6,uint param_7,uchar **param_8,uint *param_9,bool *param_10)

{
  undefined8 uVar1;
  
  if (((param_1 != (gatekeeper_device *)0x0) && (param_4 != (uchar *)0x0)) &&
     (param_6 != (uchar *)0x0)) {
    uVar1 = Verify((TrustKernelGateKeeperDevice *)param_1,param_2,param_3,param_4,param_5,param_6,
                   param_7,param_8,param_9,param_10);
    return uVar1;
  }
  return 0xffffffea;
}
```



### OpenTEESession Execution flow 

The main purpose of this is to flow like such to create an Entrypoint for the application to work, in our case it would be to verfiy passwords of some kind. 

  TEEC_Opensession           <---------------------->		TA_OpenSessionEntryPoint 
  The first stage is to get the context:

```c++ 
  pTVar1 = this + 0xa0;
  iVar2 = TEEC_InitializeContext(0,pTVar1);
```
if the context is valid TEEC_OpenSession is called. If the ID is not 0 the session will fail.
The data taken in are 

```c++
  iVar2 = TEEC_OpenSession(pTVar1,this + 0x1a8,&DAT_00100f90,0,0,0,0);
  *(int *)(this + 0x98) = iVar2;
  while (iVar2 != 0) {
    __android_log_print(6,"GatekeeperHAL"," OpenSession failed with 0x%08x\n",iVar2);
    sleep(1);
    iVar2 = TEEC_OpenSession(pTVar1,this + 0x1a8,&DAT_00100f90,0,0,0,0);
    *(int *)(this + 0x98) = iVar2;
  }
  __android_log_print(4,"GatekeeperHAL","Open session successfully\n");
  return 0;
}
```
Now if we remeber what TEEC_OpenSession takes in this may help get a undertanding of what is going on here:
```c++

 iVar2 = TEEC_OpenSession(pTVar1,this + 0x1a8,&DAT_00100f90,0,0,0,0);
 
 iVar2 = TEEC_OpenSession(session,destination + connectionMethod,&connectionData,0,0,0,0);
 ```


