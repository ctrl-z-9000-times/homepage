# A Database for Neural Simulations

## Introduction

This document seeks to design a database suitable for the internal data
structures of NEURON and of neural simulators in general. A database is a
software tool for organizing and managing data.

### The State of the Art

Currently NEURON and all other state of the art neural simulators are written in
an object oriented programming style. Data is held in arrays of structures or
classes which are connected together by pointers. They use the programming
languages built-in data structures and algorithms. Their data structures are
specified by their source code, which works well for an immediate
implementation, but also forces them to use the syntax and semantics of the
programming language which they write in.

### The case for using a Database

 * One benefit of using a database is that it gives you a way to think about
   data in the abstract. By collecting all of your data into a single place you
   can see all of the commonalities between the different pieces of data in
   your simulation. To that end, this document seeks to gather all requirements
   and constraints for the data structures inside of your simulations.

 * Another benefit of using a database is that you can consolidate duplicate
   pieces of software into a shared code-base. You can write software tools
   which do more things with fewer lines of code and with almost no redundant
   code duplication (at least related to memory management). Fixes and new
   features can be enabled and applied to every datum from a central location.

 * Finally, if you make your own database then you can make it to bespoke to
   suit the needs of your programs.

### Innovations

Most of aspects of this design are not novel. Rather, they came about as
solutions to common problems that arose in the course of implementing neural
simulators. What is novel is the combination of all of these pieces to form a
cohesive set of tools, where all of the pieces work together with synergy but
still remain flexible.

## User Profiles

The database will be used by two groups of people: first by programmers and then
by the end users of their programs. The database will have two API's, one for
each user group.

### Profile of a Computer Programmer

* Has technical experience and ability:
    - Reads the documentation.
    - Learns the details of a given computer system.
    - Debugs unexpected issues, can read a stack backtrace.
    - Trusted with your programs private variables.
* Wants computer speed and memory performance.

### Profile of the End User

* Minimal technical experience:
    - Wants easy to use.
    - Not good at debugging software.
    - Unwilling to learn new 'programming paradigms'. Expects computer systems
      to conform to their existing mental models of how things should work.
* Wants high quality results that can be directly published.

### Two API's

The programmers API allows access to the data arrays.

The end users API presents an easy to use interface:
 * Python. It should use all of pythons built-in features:
    + Classes
    + Docstrings
    + Properties for setter/getter methods

## Terminology

This database is tailored for running physics simulations, and to that end it
has special words for dealing with the physical things which exist in your
simulation.
 * An **Entity** is a specific object that exists in your simulation.
 * An **Archetype** defines a type of Entity, and each Entity is an instance of
   exactly one Archetype.
 * A **Component** is piece of data which is attached to an Entity.

There is a clear correspondence between these terms and the more familiar object
oriented terminology of C, C++, Python, etc. For example consider the following
python program:

```python
class Neuron:
    def __init__(self):
        self.voltage = -70 # mV

my_neuron = Neuron()
```

Now let's describe this program using the database terminology:
 * "Neuron" is an **Archetype**.
 * "voltage" is a **Component**.
 * "my_neuron" is an **Entity**.

## Requirements

### Creating Schema
 * Programmers can create Archetypes and Components. Although typically these
   will only be created at the start of the program, it would be convenient to
   add things to the database while the program is running.

### Components

There are many different types of components for representing different and
specialized data structures and algorithms.

#### Attributes

 * An attribute is a simple piece of data attached to an Entity.
 * The data type can be anything, including arbitrary user defined data types.
   The most common data types will be floating-point and pointer types.
 * Attributes must keep all of their data in a single contiguous array each
   containing exactly one attribute. This is essential for fast processing of
   data.

#### Global Constants

 * This is a special case of an attribute, it's like an attribute except that
   all Entities see the same constant value.

#### Sparse Matrixes
 * Compressed Sparse Row (CSR) matrixes are a way for Entities to have lists of
   pointers to other Entities.
   - For example you could have a sparse matrix of "synapse_weights" which gives
     each presynaptic neuron a list of postsynaptic dendrites and associated
     weights.
   - Another example to store the electrical resistivity between adjacent
     segments using a sparse matrix.

#### Error Checking

Each component can be configured to check for common issues.
 * All components that can contain floating-point numbers will have the
   following optional checks on their data:
    + Check for NaN.
    + Bounds check `min <= value <= max`
 * All components that can contain a pointer will have an optional check for
   NULL pointers.
 * By default, all checks should be disabled.
 * Checks should be configured when the component is first specified/created.
 * The database will provide an API method for running all of the checks as well
   as checking specific components.

### Entities

 * Any user may create and destroy entities at any time.
 * Any user may access the component data at any time.

#### Entity Handles

EntityHandles are special code objects that the database uses to represent a
specific entity.
 * When the end user creates an entity they get an EntityHandle in return.
 * EntityHandles can destroy their entity.
 * EntityHandles can access data associated with their entity.
 * EntityHandles are persistent. They are valid until the user destroys either
   the underlying entity or the handle.
 * EntityHandles should be easy to use. For this API: it is acceptable to trade
   off poor computer performance for a more polished end-user experience.

#### Entity Storage and Movement
 * Entities are stored in contiguous zero-indexed arrays, where each array
   contains all entities of their archetype.
 * Holes are not allowed in the entity arrays. Therefore when the user deletes
   an entity from the middle of the list of entities, then a different entity
   is moved to fill in the hole and all pointers to the moved entity are
   updated to point to the new address.
   + This is possible because the database knows the memory location of every
     pointer and every EntityHandle.
   + This operation can be done efficiently if all entities are destroyed in
     batches, as opposed to one at a time. The algorithm to move entities reads
     and writes every pointer in the database exactly once, and can move any
     number of entities simultaneously.
      - Programmers need to batch up their operations.
      - Linear run time.
   + The alternative implementation is a dead/alive mask over the entity arrays.
      - The programmers to need to understand how to use the alive mask.
      - Linear memory usage.
 * Entities can also be reordered for optimal data access patterns. This is
   helpful when data access patterns span multiple archetypes.
   + For example, ion channels have a pointer to the neuron segment where they
     are inserted. The channels should be sorted by that pointers literal
     value, because usually when you iterate through a list of ion channels
     you're also going to want to access the data for the segments which they
     are inserted into. When the entities are sorted by their pointers and you
     iterate through the list of entities accessing each pointers' data, then
     you are accessing all of the data in a single sorted sweep through the
     underlying data.

#### Pointers

A pointer is a way to refer to an Entity.
 * Pointers are for the programmers API. Programmers can use pointers, end users
   may **not** use pointers. End users must use the EntityHandle instead.
 * Pointers are represented as indexes into the entity arrays. This is important
   for using a structure-of-arrays memory layout. It also allows pointers to be
   represented in 32 bits instead of 64 bits.
    - NULL is represented by the maximum value instead of zero, since zero is a
      valid index.
 * Programmers may use pointers that are stored in the database, with the
   limitation that pointers are only valid until the next time the database
   gains control of program execution (at which time the database may decide to
   move entities and update all pointers). The database will provide a way to
   make persistent EntityHandles out of the transient pointers.

#### Destroying Entities

Pointers to destroyed entities are invalid and must be either overwritten or
destroyed. Components containing pointer values can be configured to either
allow or disallow NULL pointers.
   + If NULL pointers are allowed then pointers to destroyed entities are
     replaced with NULL pointers.
   + If NULL pointers are disallowed then the entities that contain them are
     automatically destroyed. This can trigger a chain reaction of
     destruction.
      - For example, this is useful for attaching ion-channels to a segment of a
        neurons membrane because then when you destroy the segment it
        automatically destroys all attached ion-channels.

#### Sets of Entities

Programmers will need persistent references to large numbers of entities.
 * Using a large number of EntityHandles could be inefficient since each handle
   is its own discrete python object.
 * Creating a new archetype with a pointer attribute is cumbersome, visible
   throughout the database, and it has different semantics than regular python
   code.

To meet this need: the database could implement efficient collection types for
Entities.
 * Set of entities must be of a single homogeneous Archetype.
 * Backed by either Arrays or Hash-Sets.
 * Applies operations to all contained elements.

However, I would recommend deferring the design and implementation of this idea
until after the rest of the database is implemented and programmers start
trying to use it, at which point it will become more clear what faculties the
database lacks regarding collections of entities.

### Documentation

As the central repository for the data, it makes sense to also store any
descriptions of the data in the database too.
 * The programmer can attach user-facing documentation to any Archetype or
   Component. The documentation can be as simple text string.
 * The database will render user-facing documentation of itself, showing its
   database schema, and including all attached documentation.
 * Physical units are a special type of documentation, and the database treats
   them with a special case.

### Introspection

The database will provide an API for inspecting its database schema and all
associated meta-data, at run time.
 * The primary motivation for introspection is creating user interfaces
   (UI's). UI programs need to know how the underlying database is organized in
   order to effectively present it to the user. By reading the database schema
   at run time the UI can be much more flexible and easier to write & maintain.

### Hardware Memory Spaces

The database can utilize multiple memory spaces.
 * CPU: host only.
 * GPU: connected to host, openCL or CUDA?
 * Other. The database should be implemented in such a way that it is possible
   to add other hardware platforms.

TODO: how is this specified? by default where do things live? what kind of
controls does the programmer/user have over it?

### Other Ideas

#### Grids of Entities

An Archetype could define a regular grid to place Entities on and provide tools
for efficiently working with grids of Entities. For example this could be used
to implement extracellular diffusion at a course granularity.

#### Temporary Buffers

It might be nice to have your working data be managed by the database, but not
persistently stored. The programmer would be responsible for free'ing it when
they're done with it, or else it would just sit around like permanent data.

#### Spatial Partitioning Structures
 - For performing fast nearest neighbor searches.
 - Requires valid/invalid book keeping.

#### Linear Systems
 - Different solver algorithm than NEURON, might be faster but its not as accurate.
 - Requires valid/invalid book keeping.
