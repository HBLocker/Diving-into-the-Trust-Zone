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
