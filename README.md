# FreeRTOS-Pooling-Server
## Implement polling server for FreeRTOS

```
include/FreeRTOS.h - We've added a declaration for Macro portIS_ENDED_API (Line 509)
portable/MSVC-MingW/portmacro.h - We've added a declaration for Macro portIS_ENDED (Line 99)
	#define portIS_ENDED(pxTopOfStack) vIsEnded(pxTopOfStack)
	
portable/port.c - We've added a function "vIsEnded"
```

### In file tasks.c:

	// Update of struct tskTCB. We've added:
	```
		unsigned long capacityC; // Task capacity
		unsigned long periodT;   // period of task
		unsigned long start;	  // Tick in which task has started
		unsigned long endT;   // Tick in which task has ended
		
		pdTASK_CODE pxTaskCode;   // Pointer to function that task should execute
		void *vParam;			  // Parameters that are passed to function
		
		
	// Define struct aptskTCB. This struct describes aperiodic tasks.
	
		typedef struct aperiodicTaskControlBlock {
			unsigned long capacityC;	// capacity - execution time of aperiodic task, but it is not real time.
										// It is number how many CPU ticks it needs to be executed.
			unsigned long durationC;	// How long the task is executed
			pdTASK_CODE pxTaskCode;		// PPointer to function that task should execute
			void *vParam;				// Parameters of function
		} aptskTCB;
	

	// Added some variables on lines 15 to 170
	
		PRIVILEGED_DATA static unsigned long serverCapacityC = 0;			// Max capacity of server
		PRIVILEGED_DATA static unsigned long serverCapacityCCurrent = 0;	// Current capacity of server
		PRIVILEGED_DATA static unsigned long serverPeriodT = 0;			// Server period
		PRIVILEGED_DATA static unsigned long serverWorkingTicks = 0;			// How much ticks has server been working in current period
		PRIVILEGED_DATA static unsigned long serverBeginWorking = 0;			// In which ticks scheduler switch on server
		PRIVILEGED_DATA static unsigned long serverLastTick = 0;			// Last server tick
		
		PRIVILEGED_DATA static unsigned long serverLastPeriod = ULONG_MAX;	// Time when server have been last time updated
		
		PRIVILEGED_DATA static tskTCB *serverTCB = 0;						// Pointer to TCB of server
		
		PRIVILEGED_DATA static xList pxReadyAperiodicTasksLists;		// Tasks that are ready for server to be executed i.e. tasks that server have taken
		PRIVILEGED_DATA static xList pxWaitingAperiodicTasksLists;		// All aperiodic tasks that are waiting for server to be executed
		PRIVILEGED_DATA static int areAperiodicListsInitialised = 0;	// Additional variable that we use to initialize above 2 lists
				
		

	// Function vTaskStartScheduler(Linije od 110 do 1115) is key to start server. In fact it calls just xTaskCreate with correct parameters.
	
		static int runned = 0;
		if (runned == 0) { // Start server just one time
			serverTCB = (tskTCB*) malloc(sizeof(tskTCB));
		
			// Creating periodic task which function is prvServerTick.
			xTaskCreate(prvServerTick, (signed char * ) "Server",
					configMINIMAL_STACK_SIZE, NULL, uxPriority, &serverTCB, kapacitetC, periodaT);
			// Inicialize global parameters
			serverCapacityC = kapacitetC;
			serverCapacityCCurrent = kapacitetC; // Server at start have max capacity
			serverPeriodT = periodaT;
		}
		runned++;
	
	
	// Declared function prvServerTick on line 425. Ova f-ja izvrsava kod servera
	
		static void prvServerTick();
		
	
	// Implement function prvServerTick on line 1894. This function is execudet in server.
	
		void prvServerTick() {
			while (1) {
				printf("%lu (Server Message) Starting Next Task\n", xTaskGetTickCount());
				fflush(stdout);
		
				// Pick first task that is ready i.e. the one that is first in readyAperiodicTaskLists
				xListItem *curr = (&((&pxReadyAperiodicTasksLists)->xListEnd))->pxNext;
		
				//go through all ready tasks and execude them in order
				int i = 0;
				unsigned n = pxReadyAperiodicTasksLists.uxNumberOfItems;
				for (; i < n; ++i) {
					xListItem *next = curr->pxNext;
					aptskTCB *currentTCB = curr->pvOwner;
					(*currentTCB->pxTaskCode)(currentTCB->vParam);
		
					// When it's finished, remove him from ready list
					vListRemove(curr);
					curr = next;
				}
				// When you have finished everything - suspend me
				printf("%lu (Server Message) Server is suspended [No more tasks]\n",
						xTaskGetTickCount());
				fflush(stdout);
				vTaskSuspend(serverTCB);
			}
		}
		
	
	// Implemented function xTaskCreateAperiodic in taks.c (line 607). But it is declared in task.h (line 1319). This function just add tasks in aperiodicTaskList
	
		void xTaskCreateAperiodic(pdTASK_CODE pvTaskCode, void *pvParameters,
			unsigned long capacityC) {
		
			// Allocate new aptskTCB
			aptskTCB *newTCB = (aptskTCB*) malloc(sizeof(aptskTCB));
			newTCB->capacityC = capacityC;
			newTCB->vParam = pvParameters;
			newTCB->pxTaskCode = pvTaskCode;
			newTCB->durationC = 0;
		
			// Allocate new waiting tasks list node
			xListItem *newItem = (xListItem*) malloc(sizeof(xListItem));
			newItem->pvOwner = newTCB;
			vListInitialiseItem(newItem); // Inicialize
			// If it's this function first time called, then initialize lists for waiting and ready tasks
			if (areAperiodicListsInitialised == 0) {
				vListInitialise(&pxWaitingAperiodicTasksLists);
				vListInitialise(&pxReadyAperiodicTasksLists);
				areAperiodicListsInitialised = 1;
			}
			// Put him in lists
			vListInsertEnd(&pxWaitingAperiodicTasksLists, newItem);
		}
	
	
	// Function vTaskSwitchContext is the most important function in this project. See in code comment what does it do.
	
	
## main.c: Everithing is ours


