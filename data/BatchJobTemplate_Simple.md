# Batch Job Template - Simple Example

## Overview

This file demonstrates a simple implementation of the SysOperation Framework for creating batch jobs in Dynamics 365 Finance and Operations. The example consists of three main components:

1. **sunSimpleContract** - A data contract class that defines the parameters for the batch job
2. **sunSimpleController** - A controller class that manages the execution of the batch job
3. **sunSimpleService** - A service class that contains the actual business logic to be executed

This template follows best practices for batch job implementation in D365 F&O, including proper error handling with retry logic for deadlocks and update conflicts.

```xpp
/// <summary>
/// Example data contract for use with the SysOperationFramework
/// Goes with the sunSimpleController and sunSimpleService
/// </summary>
[DataContract]
[SysOperationAlwaysInitialize]
final class sunSimpleContract implements SysOperationInitializable
{
    // Data contract properties are kept private and accessed through parm methods
    private int gWaitTime;
    private str gInfoText;

    /// <summary>
    /// gets/sets the wait time property
    /// </summary>
    /// <param name = "_waitTime"> the time to wait between the first and second infologs</param>
    /// <returns> wait time property</returns>
    [DataMember,
        SysOperationLabel(literalStr("Wait time")),
        SysOperationHelpText(literalStr("The wait time for the job")),
        SysOperationDisplayOrder('1')]
    public int parmWaitTime(int _waitTime = gWaitTime)
    {
        gWaitTime = (_waitTime > 20000) ? 20000 : _waitTime;
        return gWaitTime;
    }

    /// <summary>
    /// gets/sets the infoText property
    /// </summary>
    /// <param name = "_infoText"> the text to show in the job</param>
    /// <returns> the infoText property</returns>
    [DataMember,
        SysOperationLabel(literalStr("Info text")),
        SysOperationHelpText(literalStr("The info text for the job")),
        SysOperationDisplayOrder('1')]
    public str parmInfoText(str _infoText = gInfoText)
    {
        gInfoText = _infoText;
        return gInfoText;
    }

    /// <summary>
    /// Initializes the default values of the job parameters
    /// </summary>
    public void initialize()
    {
        gWaitTime = 5000;
        gInfoText = "Default job text";
    }

}
```

```xpp
/// <summary>
/// Simple SysOperationFramework controller
/// Works with the sunSimpleService and sunSimpleContract
/// </summary>
final class sunSimpleController extends SysOperationServiceController
{
    /// <summary>
    /// Override of the base new method
    /// passes our service class and method
    /// </summary>
    protected void new()
    {
        super(classStr(sunSimpleService),
                methodStr(sunSimpleService, process),
                SysOperationExecutionMode::Synchronous);
    }

    /// <summary>
    /// default caption for the batch job dialog
    /// </summary>
    /// <returns> the caption for the dialog</returns>
    public ClassDescription defaultCaption()
    {
        return "@JJ:sunSimple";
    }

    /// <summary>
    /// Constructor for our controller
    /// </summary>
    /// <param name = "_executionMode"> The execution mode of the job</param>
    /// <returns> A new controller instance with the proper execution mode</returns>
    public static sunSimpleController construct(SysOperationExecutionMode _executionMode = SysOperationExecutionMode::Synchronous)
    {
        sunSimpleController controller;
        controller = new sunSimpleController();
        controller.parmExecutionMode(_executionMode);
        return controller;
    }

    /// <summary>
    /// Entrypoint from the job's menu item
    /// </summary>
    /// <param name = "_args"> Args from menu item</param>
    public static void main(Args _args)
    {
        sunSimpleController controller;
        controller = sunSimpleController::construct();
        controller.parmArgs(_args);
        controller.startOperation();
    }

}
```

```
/// <summary>
/// Simple SysOperationFrameworkService
/// Works with the sunSimpleContract and the sunSimpleController
/// </summary>
final class sunSimpleService extends SysOperationServiceBase
{
    /// <summary>
    /// Processing method for the job
    /// Infologs some text, waits a set time, and infologs more text
    /// </summary>
    /// <param name = "_contract"> the data contract containing the batch job parameters</param>
    public void process(sunSimpleContract _contract)
    {
        #OccRetryCount
        int waitTime = _contract.parmWaitTime();
        str infoText = _contract.parmInfoText();
        try
        {
            ttsbegin; //Not needed in this situation, but relevant for real jobs
            info(strFmt("Starting: %1 at %2",infoText,datetime2Str(DateTimeUtil::getSystemDateTime())));
            sleep(waitTime);
            info(strFmt("Ending: %1 at %2", infoText,datetime2Str(DateTimeUtil::getSystemDateTime())));
            ttscommit;//Not needed in this situation, but relevant for real jobs
        }
        catch(Exception:: Deadlock)
        {
            if(xSession::currentRetryCount() >= #RetryNum)
            {
                throw Exception::Deadlock;
            }
            else
            {
                retry;
            }
        }
        catch(Exception::UpdateConflict)
        {
            if(appl.ttsLevel() == 0)
            {
                if(xSession::currentRetryCount() >= #RetryNum)
                {
                    throw Exception::UpdateConflictNotRecovered;
                }
                else
                {
                    retry;
                }
            }
            else
            {
                throw Exception::UpdateConflict;
            }
        }
    }

}
```