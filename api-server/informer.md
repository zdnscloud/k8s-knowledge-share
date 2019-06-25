# Informer
1. Overview
  * Watch resource event
  * Based on event to maintain local resource cache
  * Allow to register event handler for resource change
  * Provide read interface to local cache
2. Error handling
  * API server will drop long connection(watch), informer will rewatch
  * Support resync logic which will reconciliation between the in-memory
    cache and the business logic by invoke the add/update call back.
3. Efficiency
  * to unburden the API server, one GVK should has one informer
  * shared informer factory allows to share informer in an application 
  * informer use WaitForCacheSync to wait for List calls to API server finish
  * retelimit queue is used to avoid push too many events to a slow controller
