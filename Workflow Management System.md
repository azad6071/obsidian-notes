Workflow Management System

We'll have some tools which the user can use for making the workflow.
These tools are input/output device. They can be used for generating the output for a given input.
3 tools
    LLM Text/Json
    LLM Image
    LLM Video
    Send over Gmail. 
    SMS

Every tool
    Input Schema
    ↓
    Execution
    ↓
    Output Schema

So For images/vedio we can have the url or s3 bucket where it's stored.
Workflows can be linked between each other. 
Workflow Builder - Step by Step (Max 3 Steps-extendable later) - Build Workflows composed of given tools
Workflow Run - Execute the Workflow successfully. 
Tool execution can be async and could take time. 
Long running tasks should be asynchronous and non-blocking. 
So they will return a job_id and will continue their execution.
Node binding
Auto Recovery (Failure management), Recovery should idealy retry only the step it failed

Making a workflow can be exposed via an api, where user can give input as text as payload to the api.
So user makes a workflow. Based on that we should be able to expose api. 
So there would be new api's for new workflow, basis the tools that we have choosen our api payload would differ.
Workflows can be stored 
Parse/route the input to the first tool.
Propagate data through the chain.
It should have the if-else capability for workflow design. 
For example, 
    1. Contact payload can have mobile number or email. Basis that we can trigger one of the tool
    2. We may need to generate image/video
These can be achieved by supporting tools like drag-drop for conditional rendering 
of workflow node and api payload generation
I think deciding variable can be part of payload which should be json. Decide for api design

7. Scaling - N Nodes, `Dependent ones, Condition based.
