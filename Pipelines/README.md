# Origin
This work is based on the solution found ![here](https://github.com/NDecision/Banzai)

## Pipelines - Your Simple Pipeline Solution
Pipelines is an easy .Net pipeline solution that contains composable nodes for constructing simple and complex pipelines.  
Yes, there is TPL Dataflow and it's really cool, but Pipelines provides something easy that solves the 80% case of simple
asynchronous pipelines in business applications.  It's important to understand that Pipelines is primarily about async, not parallel. Although it does take
a DegreeOfParallelism when executing against an enumerable of subjects.  
Pipelines is optimal for setting up business pipelines which have operations that benfit from async, such as external web service calls or Database/file I/O operations.
Of course it can be used to organize code regardless of external I/O, but it's performance advantage is primarily based on async and the pipeline pattern is a good way 
to force some organizational constraints on processing pipelines, such as applying rules or transformations to a subject.

## Basic Concepts
Flows are composed from nodes of which there are a few types explained below.  Nodes are [composed into simple or complex flows](#building-flows) and then 
[executed on the flow subject](#running-nodes).  All flows accept a Subject Type (T). This the type of the subject that is acted upon by the flow.  
All Node methods that are either overridden or provided via a function accept an [ExecutionContext](#executioncontext).  
All node executions return a [NodeResult](#noderesult).

````csharp
    //Create a pipeline node
    var pipelineNode = new PipelineNode<TestObjectA>();

    //Add a couple of child nodes to the pipeline that do things
    pipelineNode.AddChild(new SimpleTestNodeA1());
    pipelineNode.AddChild(new SimpleTestNodeA2()); 

    //Create the subject and execute the pipeline on it.
    var subject = new TestObjectA();
    NodeResult result = await pipelineNode.ExecuteAsync(subject);
````

### Basic Nodes
These are the nodes that contain functionality that runs against the subject of the pipeline.  Basically, this is where most of your code goes.

<b>Node/INode</b> - The simplest node type.  This is overridden to provide functionality.  

<b>FuncNode/IFuncNode</b> - Inherits node and allows functions to be assigned to provide functionality.

#### Node/FuncNode Usage
  * Override PerformExecute/PerformExecuteAsync to perform operations on the subject.

  * Override ShouldExecute to determine if the node should execute.

  * <b>PerformExecuteFuncAsync</b> - Property that accepts a function to perform on the subject.

  * <b>ShouldExecuteFuncAsync</b> - Property that accepts a function to determine if the node should be executed.  Not strongly typed.

  * <b>AddShouldExecute</b> - Extension method to set ShouldExecuteFunc in a strongly typed manner. Has to be done this way to enable variance.


Example of overriding to provide functionality
````csharp
    public class SimpleTestNodeA1 : Node<TestObjectA>
    {
        public override Task<bool> ShouldExecuteAsync(ExecutionContext<TestObjectA> context)
        {
            return Task.FromResult(true);
        }

        protected override Task<NodeResultStatus> PerformExecuteAsync(ExecutionContext<TestObjectA> context)
        {
            context.Subject.TestValueString = "Completed";
            return Task.FromResult(NodeResultStatus.Succeeded);
        }
    }
````
Example of providing functionality via functions
````csharp
    var node = new Node<TestObjectA>();

    node.AddShouldExecute = context => Task.FromResult(context.Subject.TestValueInt == 5);
    node.ExecutedFuncAsync = context => 
        { 
          context.Subject.TestValueString = "Completed"; 
          return Task.FromResult(NodeResultStatus.Succeeded); 
        };
````
### Multi Nodes
The following nodes allow you to organize and run other nodes together. These nodes are sealed and are intended for direct use.
If you wish to create a custom MultiNode, reference [Abstract Multi Nodes](#abstract-multi-nodes) below.

<b>PipelineNode/IPipelineNode</b> - Runs a group of nodes serially on the subject.  This will be the root node of most flows.

![Pipeline Node](/img/PipelineNode.png?raw=true "Pipeline Node")
````csharp
    var pipelineNode = new PipelineNode<TestObjectA>();

    pipelineNode.AddChild(new SimpleTestNodeA1());
    pipelineNode.AddChild(new SimpleTestNodeA2());

    var testObject = new TestObjectA();
    NodeResult result = await pipelineNode.ExecuteAsync(testObject);
````
<b>GroupNode/IGroupNode</b> - An aggregation of nodes that are run on a subject using the asyncrhonous Task.WhenAll pattern.

![Group Node](/img/GroupNode.png?raw=true "Group Node")
````csharp
    var groupNode = new GroupNode<TestObjectA>();

    groupNode.AddChild(new SimpleTestNodeA1());
    groupNode.AddChild(new SimpleTestNodeA2());

    var testObject = new TestObjectA();
    NodeResult result =  await groupNode.ExecuteAsync(testObject);
````
<b>FirstMatchNode/IFirstMatchNode</b> - An aggregation of nodes of which the first matching its ShouldExecute condition is run on the subject.

![First Match Node](/img/FirstMatchNode.png?raw=true "First Match Node")
````csharp
    var matchNode = new FirstMatchNode<TestObjectA>();

    var firstNode = new SimpleTestNodeA1();
    firstNode.ShouldExecuteFunc = 
        context => Task.FromResult(context.Subject.TestValueInt == 0);
    matchNode.AddChild(firstNode);

    var secondNode = new SimpleTestNodeA2();
    secondNode.ShouldExecuteFunc = 
        context => Task.FromResult(context.Subject.TestValueInt == 1);
    matchNode.AddChild(secondNode);

    var testObject = new TestObjectA();
    NodeResult result = await matchNode.ExecuteAsync(testObject);
````
### Abstract Multi Nodes
These nodes are synonymous with the above nodes, but are abstract and intended to serve as a base class for custom implementations.
If you wish to create your own multi-node, inherit from one of these.  These abstract classes contain all of the functionality 
of the corresponding concrete multi-nodes (the concrete multi-nodes directly inherit from these).

<b>PipelineNodeBase/IPipelineNodeBase</b> - Base for concrete pipelines.

<b>GroupNodeBase/IGroupNodeBase</b> - Base for concrete group nodes.

<b>FirstMatchNodeBase/IFirstMatchNodeBase</b> - Base for concrete first match nodes.

### Transition Nodes
In some cases, you need to transition during pipeline execution from one subject type to another.  To accomplish this, use the TransitionNode or TransitionFuncNode.  
These nodes take a source type and a destination type and allow you to perform any necessary transitioning from source to destination before execution, 
and allow the original source to be updated after execution. Transition nodes take a child node to execute after transition occurs.  
If the source reference has been changed during the result transition, a [ChangeSubject](#changing-the-subject) call is automatically made to set the correct subject on the ExecutionContext. 
The aggregate result and any exceptions are passed back to the source node that called the transition node.

![Transition Node](/img/TransitionNode.png?raw=true "Transition Node")

<b>ChildNode</b> - Assigns a child node that is executed after the transition to the destination type occurs.

<b>TransitionSourceAsync</b> - Transitions the source to the destination type.

<b>TransitionResultAsync</b> - Transitions the source after node execution based on the destination node results.

<b>TransitionSourceFuncAsync</b> - In TransitionFuncNode, allows assignment of source to destination transition function.

<b>TransitionResultFuncAsync</b> - In TransitionFuncNode, allows assignment of post-run source transition function.
````csharp
    public class SimpleTransitionNode : TransitionNode<TestObjectA, TestObjectB>
    {
        protected override TestObjectB TransitionSource(ExecutionContext<TestObjectA> sourceContext)
        {
            return new TestObjectB();
        }

        protected override TestObjectA TransitionResult(ExecutionContext<TestObjectA> sourceContext, NodeResult result)
        {
            return sourceContext.Subject;
        }
    } 
````
Or
````csharp
    var pipelineNode = new PipelineNode<TestObjectA>();

    pipelineNode.AddChild(new TransitionFuncNode<TestObjectA, TestObjectB>
    {
        ChildNode = new SimpleTestNodeB1(),
        TransitionSourceFunc = ctxt => new TestObjectB()
    });
````
### Should Execute Blocks
In some cases, you would like to create a reusable rule to determine if a node should execute, but keep the logic independent
of the node itself.  In this case, there is the ShouldExecuteBlock.  This block provides one method to implement, ShouldExecuteAsync, 
which returns a true or false:
````csharp
    public class ShouldNotExecuteBlockA : ShouldExecuteBlock<TestObjectA>
    {
        public override Task<bool> ShouldExecuteAsync(IExecutionContext<TestObjectA> context)
        {
            return Task.FromResult(false);
        }
    }
````
To use the ShouldExecuteBlock, call the AddShouldExecuteBlock method of the Node:
````csharp
    var node = new FuncNode<TestObjectA>();
    node.AddShouldExecuteBlock(new ShouldNotExecuteBlockA());
````

## Running Nodes
In order to run a node, you can call one of the following methods, which exist on all nodes:

<b>ExecuteAsync</b> - Executes the node given either a subject (the object your are sending through the flow) or an [ExecutionContext](#executioncontext) that contains the subject.

<b>ExecuteManyAsync</b> - Just like ExecuteAsync, but accepts an enumerable of subjects. Executes the tasks asynchrounously, so several could run simultaneously.
The results are aggregated and returnd via a single [NodeResult](#noderesult)

<b>ExecuteManySeriallyAsync</b> - Executes many, but awaits each subject so that they are guaranteed to execute serially.

### ExecutionContext
The execution context flows through all nodes in the flow.  The execution context contains options for running the flow as well as the
instance of the subject that the flow is executed on.  The type of the ExecutionContext is covariant, so it will allow ExecutionContexts based on 
inherited types to utilize a flow designed for a base type.

<b>Subject</b> - This is the main subject instance that all nodes in the flow operate on.  If it is necessary to change the subject reference, use the ChangeSubject() method of the 
ExecutionContext to do so.

<b>State</b> - The execution context also contains a dynamic State property that can be used to 
flow any random state needed for the workflow.  Any node in the flow can update the state or add dynamic properties to the state.
````csharp
    var context = new ExecutionContext<object>(new object());
    context.State.Foo = "Bar";
````
<b>GlobalOptions</b> - ExecutionOptions that are set in the context and applied to every node.

<b>ParentResult</b> - The root result of this node execution and all of its children.

<b>CancelProcessing</b> - Cancels any further processing of the flow.  This only cancels the current subject iteration of an ExecuteMany.

#### ExecutionOptions
The execution options impact the behavior of node execution. 
 
<b>ContinueOnFailure</b> - Continue execution of other nodes under the parent if this node fails. Defaults to false.

<b>ThrowOnException</b> - Should a node execution exception throw or register as a failure and store the exception in the NodeResults exception property.
Defaults to false.

<b>DegreeOfParallelism</b> - The maximum number of parallel operations that are used to process the subjects when calling [ExecuteManyAsync](#running-nodes).

### NodeResult
When a node executes, it returns a NodeResult and also exposes the current or last NodeResult in the Result property.  
The NodeResult will contain:

<b>NodeResultStatus</b> - Represents the status of this node.  If this is a parent node, it represents a rollup status of all child nodes.

<b>Subject</b> - A reference to the subject, stored as an object.

<b>GetSubjectAs</b> - Allows a strongly typed subject to be returned.

<b>ChildResults</b> - A collection of child result nodes corresponding to the current node's children.

<b>Exception</b> - An Exception if any exception occurred during execution of the node.

<b>GetFailExceptions()</b> - This method aggregates all the exceptions on the failure path of the current node and returns them as an IEnumerable&lt;Exception&gt;.  
This includes any exception from nodes that contributed to a failure status of the current.  It's important to know that if a PipelineNode is set to ContinueOnFailure 
and some of the nodes succeed, the PipelineNode will have a status of PartiallySucceeded and as such will not have failures added to this collection.

#### NodeResultStatus
Each node returns a result that contains a status when run.  The status will be one of the following:

  * NotRun - Node has not been run
  * SucceededWithErrors - Node is flagged as succeeded, but some error occurred.  Typically indicates that a subnode failed but "ContinueOnError" was set to true.
  * Succeeded - Node Succeed
  * Failed - Node reported a failure or an exception was thrown during the execution of the node.

### NodeRunStatus
Represents the run status of the node.  The status will be one of the following:
  * NotRun - The node has not been run
  * Running - The node is in process
  * Completed - The node has completed
  * Faulted - The node faulted (threw an exception)

## Building Flows
There are multiple ways in which flows can be built up.

### Manually Constructing Flows
Any node can be manually added to the Children collection of any Multinode (Pipeline, Group, FirstMatch) using the AddChild or AddChildren methods.
Multinodes can be added to other multinodes, allowing a node tree to be built.
````csharp
    var pipelineNode = new PipelineNode<TestObjectA>();

    pipelineNode.AddChild(new SimpleTestNodeA1());
    pipelineNode.AddChild(new SimpleTestNodeA2());
````
Alternately, if the children are known at design time, the children may simply be added in the parents constructor.  
However, this approach will be problematic if the child nodes have dependencies injected. 

### Injecting Child Nodes
If nodes are registerd with the DI container (See [Registering Nodes](#registering-nodes)), they can be injected into the 
constructor of any node.
````csharp
    public class Pipeline1 : PipelineNode<TestObjectA>
    {
        private ISimpleTestNode _child1;

        public Pipeline1(ISimpleTestNode child1)
        {
            _child1 = child1;
        }
    }
````

### Using FlowBuilder
FlowBuilder allows you to build complex workflows with a simple fluent interface.  Complex flows can be constructed by 
adding both nodes and subflows.  Once a flow is registered, it can be accessed from the container or via the INodeFactory.

#### Methods
<b>CreateFlow</b> - Initiates flow creation and returns a FlowBuilder for the flow.

<b>AddRoot</b> - Adds a root node to a flow.  Returns a reference to a FlowComponentBuilder for the root node.

<b>AddChild</b> - Adds a child to the current node and returns the same FlowComponentBuilder for the current node (not the child).

<b>AddFlow</b> - Allows the addition of one flow as a child of the current node.  Allows for the creation of common flows that can be added to other flows.

<b>ForChild</b> - Changes the FlowComponentBuilder context to the specified child node.

<b>ForChildFlow</b> - Changes the FlowComponentBuilder context to the specified child flow node.

<b>ForLastChild</b> - Changes the FlowComponentBuilder context to the last child added to the current context.

<b>ForParent</b> - Changes the FlowComponentBuilder context to the parent of the current node.

<b>SetShouldExecuteBlock</b> - Sets the type of the ShouldExecuteBlock to apply to running this node.

<b>SetShouldExecute</b> - Allows a ShouldExecute function to be set on the fly when building a flow with flowbuilder.

<b>Register</b> - Must be called to indicate the flow definition as been completed and to register the flow with the container. 
````csharp
    var flowBuilder = new FlowBuilder<object>(new AutofacFlowRegistrar(containerBuilder));

    flowBuilder.CreateFlow("TestFlow1")
      .AddRoot<IPipelineNode<object>>()
      .AddChild<ITestNode2>()
      .AddChild<IPipelineNode<object>>()
        .ForChild<IPipelineNode<object>>()
        .SetShouldExecute(ctxt => (ctxt.Subject.Foo == "bar"))
        .AddChild<ITestNode4>()
        .AddChild<ITestNode3>()
        .AddChild<ITestNode2>();
    flowBuilder.Register();

    var container = containerBuilder.Build();
````
Access it via the container:
````csharp
    var flow = container.ResolveNamed<FlowComponent<object>>("TestFlow1");
````
Or via the nodefactory:
````csharp
    var nodeFactory = container.Resolve<INodeFactory<object>>();
    var flow = (IPipelineNode<object>)factory.GetFlow("TestFlow1");
````

## Logging
By default, Pipelines will log to the Debug console in debug mode and will not log in release mode.

You must configure the logger of your choice using Microsoft.Extensions.Logging

Pipelines will log a number of Error/Info/Debug operations to the log by default.  From any INode (v1.0.7 or greater), a Logger
will be exposed for your own custom logging.  At the DEBUG logging level, node stopwatch timings for every node run will also be logged.


## Advanced Scenarios

### Changing the Subject
In some cases, you may want to switch the subject reference during part of the flow.  One example is that you call an external
service within a node and it returns a different subject reference back to you.  In this case, you want to nodes following the 
current node to recieve the new subject.  This can be accomplished by calling the ExecutionContext.ChangeSubject() method.

### Inheritance and Variance
Both ExecutionContexts and Nodes allow variance.  In short, subjects can covary with the flow argument type, while
nodes can contravary with node argument types.  What does this mean?  The examples below clarify this a little, but I believe it works as you would logically
expect inheritance to work.

#### Subject Covariance
Suppose you have the types TestObjectA and a subclass of TestObjectA called TestObjectASub:  TestObjectASub --> TestObjectA.

If you have a flow that accepts TestObjectA, it will also accept TestObjectSubA.
````csharp
    var testNode = new SimpleTestNode_For_TestObjectA();
    var testObjectASub = new TestObjectASub();

    //Add TestObjectASub to test node created for TestObjectA
    var result = await testNode.ExecuteAsync(testObjectASub);
````
#### Node Contravariance
Again suppose you have the types TestObjectA and a subclass of TestObjectA called TestObjectASub:  TestObjectASub --> TestObjectA.

If you have a MultiNode (Pipeline/Group/FirstMatch) that has a subject Type of TestObjectSubA, it will allow the addition of Nodes that work on TestObjectA.
````csharp
    var testNodeA = new Node_Typed_For_TestObjectA();
    var testNodeASub = new Node_Typed_For_TestObjectASub();

    //Pipeline is created for TestObjectASub
    var pipeline = new PipelineNode<TestObjectASub>();
    pipeline.AddChild(testNodeA);
    //Accepts node typed for TestObjectA
    pipeline.AddChild(testNodeASub);
````
## Recommended Practices

#### Subject Envelope
Create an envelope for your subject and use this as the subject passed to the flow instead of passing the naked subject.  This envelope can contain the subject
but can also contain other properties or data that is necessary for the workflow.  Favor this over sending information in the State property of the ExecutionContext.

