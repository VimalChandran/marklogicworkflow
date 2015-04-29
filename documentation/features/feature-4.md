# Feature 4 - Supporting sub processes, loops, etc.

## How it works

1. Parent process step that can fork includes a wf:branches specification in it's pipeline state options.
1. wfu:fork called with wf:branches data and, in a single transaction, creates all sub process documents, mapping data
over, including a wf:parent URI for its parent process instance
1. wfu:fork also updates parent process instance to INPROGRESS with copy of wf:branches data in properties, ready to receive
individual sub processes' status when they are complete
1. Each sub process executes as if it were a parent process. BUT data resides in parent???
1. Upon completion, wfu:complete checks to see if we're in a sub process instance, and if so updates the parent processes'
status for the named branch's element to COMPLETE
1. wfu:complete also performs check to see if all branches in parent process are now COMPLETE, and if so, sets the
wf:currentStep status to COMPLETE (thus continuing execution of the parent process).
   - This is the same mechanism that User Task uses to restart CPF processing of a parent process

#### Implementation notes

wf:branches on parent process' properties document is a top level property, NOT under wf:currentStep. This is so that
its content is visible to wfu:complete (not deleted by wfu:completeFinally), and so available to store in the audit
record by wfu:complete.

### Things still to decide

#### Algorithm for creating pipelines

1. Separate flow analysis algorithm steps through the diagram from the start event (or other initial event), and
assigns a pipeline name to each step
1. Importer must generate for each BPMN2 step one or more (a set of) CPF pipeline states, using the pipeline name from above within the state name
 - This is basically the current import algorithm, but generating its output to a map rather than a cpf pipeline XML
1. The output is then generated as a set of pipelines with the above as output, plus standard pieces (E.g. restart status transition)
1. Domain then created with all these pipelines, plus status change pipeline

Qn: Should we create 1 domain per pipeline??? I think we will have to as else there is no way to link a specific process
(or sub process) document with a specific pipeline

Qn: Public vs. Private vs. Executable: Private processes should not be imported (they're not for implementation.) Public
processes that are tagged as executable are top level executable processes, so should be imported as such. Other public
processes can only be instantiated by top level processes. This is as per the BPMN2 specification.

As a result non top-level processes should have a name and process doc location that reflects their parent's location. If
they are 'upgraded' to 'executable' (i.e. top level) then a separate version is created at the top level URI name. This
shouldn't affect the ability to find all historic versions, but will affect the ability to start the process, or list its
model, from the REST API. (Only top level executable processes should be listed for execution).

#### Handling loops

How to model multiple instances?

- Where are loop instances specified?
  - On the step's Loop characteristic?
- How are they executed?
  - If in parallel, then we need a loop id field for each branch instances, and pass this in to subprocess along with parent URI
  - If in series, need wfu:complete to check position in loop before updating parent to COMPLETE, as we may need to go
  through another loop instance

#### Data ownership

Does each subprocess receive a copy of the parent's data, OR does it have direct (and potentially parallel) access to
that data???

#### Unknown receivers

Events/signals can be received by other processes outside of this single diagram, or indeed anywhere in this diagram.
How do we want to model and manage firing child processes where the step itself doesn't know of the receiver?

Need two ways:-
1. If an intermediate step can receive signal
1. If a start event receives the signal

For the second option above, can we create a standard name across all pipelines that would allow a cts:search to be
used to find the pipelines (current executable version only) which define an initial state that matches this signal?

For the first option above we can simply search for all INPROGRESS process instances where the event/signal definition
matches this instance's settings. Use cts:search.

VALIDATION REQUIRED

## When a Fork can happen

Any time there needs to be parallel processing. Examples include:-

- Parallel Gateway with one or more potential routes
- Invoke sub process (for separation reasons and ease of importing) - may not actually run in parallel (but MAY according to BPMN2 spec)
- Where a step has one next route, but also fires an event or signal (but not a message - they are directed and modelled as content)
- Any time an event or signal is fired (as there may be multiple receivers)
