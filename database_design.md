# A Database for Neural Simulations

## Introduction

This document seeks to design a database suitable for NEURON and for neural
simulators in general.

### The State of the Art

[Explain how NEURON uses object-oriented-programming.]

Currently NEURON and all other state of the art neural simulators are written in
an object oriented programming style. Data is held in arrays of structures or
classes which are connected together by pointers. They use the programming
languages built-in data structures and algorithms without making any attempt to
extend or improve upon them.

[Explain how this is inflexible and can lead to significant code bloat,
especially when doing things which are not natively supported by the language.]

### The case for using a Database

The primary benefit of using a database is that it gives you a way to think
about data in the abstract. By collecting all of your data into a single place
you can see all of the commonalities. You can write software tools which do
more things with fewer lines of code and with almost no redundant code
duplication. New features can be enabled and applied to every datum from a
central location.

## User Profiles

The database will be used by two groups of people: first by programmers and then
by the users of their programs.

### Profile of a Computer Programmer

* Has technical experience and ability:
    - Reads the documentation.
    - Learns the details of a given computer system.
    - Debugs unexpected issues, can read a stack backtrace.
    - Trusted with your programs private variables.
* Wants computer speed and memory performance above all else.

### Profile of a Scientist

* Minimal technical experience:
    - Wants easy to use.
    - Not good at debugging software.
    - Unwilling to learn new 'programming paradigms'. Expects computer systems
      to conform to their existing mental models of how things should work.
* Wants high quality results that can be directly published.

### Terminology

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
 * Create Archetypes and Components. Although typically these will only be
   created at the start of the program, it would be convenient for the end user
   if they could add things to the database while the program is running.

### Components

There are many different types of components for reqpresenting different and
specialized data structures and algorithms.

#### Attributes

 * An attribute is a simple piece of data attached to an Entity.
 * The data type can be anything, including arbitrary user defined data types.
   The most common data types will be floating-point and pointer types.
 * Attributes must keep their data in a single contiguous array. This is
   essential for fast processing of the data.

#### Global constants

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
 * Checks should be configured when the Component is first specified/created.
 * The database will provide an API for running all of the checks as well as
   checking specific components.

### Entity Handles
 * User may Create and Destroy Entities at any time.
 * An **EntityHandle** is a special code object that the database gives to the
   user to represent an entity.
    + When the user creates an Entity they get an EntityHandle in return.
    + Can use it to destroy the entity.
    + Can use it to access data associated with the Entity.
    + EntityHandles are persistent. They are valid until the user frees them.

### Pointers
 * A pointer is the memory address of an Entity.
 * Programmers can use pointers, end users may **not** use pointers. End users
   should use the EntityHandle instead.
 * Entities are stored in contiguous zero-indexed arrays, where each array
   contains all entities of their archetype.
 * Pointers are represented as indexes into entity arrays. This is important
   for using a structure-of-arrays memory layout. It also allows pointers to be
   represented as 32 bit numbers instead of 64 bit numbers.
 * Holes are not allowed in the entity arrays. Therefore when the user deletes
   an entity from the middle of the list of entities, then a different entity
   is moved to fill in the hole and all pointers to the moved entity are
   updated to point to the new address.
 * Moving the memory locations of entity is possible because the database knows
   the memory location of every pointer. This operation can be done efficiently
   if all entities are destroyed in a batch, as opposed to one at a time.
 * Programmers may use pointers that are stored in the database, with the
   limitation that you may not save the pointers. The database will provide a
   way to make persistent EntityHandles out of the transient pointers.

### Documentation

As the central repository for the data, it makes sense to also store any
descriptions of the data in the database too.
 * The programmer can attach user-facing documentation to any Archetype or
   Component. The documentation can be as simple text string.
 * The database will render user-facing documentation of itself, showing its
   internal database schame, and including all attached documentation.
 * Physical units are a special type of documentation, and the database allow
   you to read the units associated with a component.
    + Other ideas 

### Introspection

The database will provide an API for inspecting its database schema and all
associated meta-data, at run time.
 * The primary motivation for introspection is creating user interfaces
   (UI's). UI programs need to know how the underlying database is organized in
   order to effectively present it to the user. By reading the database schema
   at run time the UI can be much more flexible and easier to write & maintain.
   In fact one of the common "software design patterns" for making graphical
   user interfaces starts with splitting your software into three parts: the
   model, the view, and the controller. The model is a database, like what we
   are building here, although typically the database which UI's use is custom
   tailored for the specific needs of graphical user interfaces.
   + For more see: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller

### Hardware and Computing Environment

 * CPU. There will always be a CPU running python connected to the computing
   evironment.
 * GPU. Sometimes there will be graphics cards connected to the computing
   environment. The database should deal with moving data to and from it.
 * Other. The database should be implemented in such a way that it is possible
   to add other hardware platforms.

TODO: how is this specified? by default where do things live? what kind of
controls does the programmer/user have over it?

### Miscellaneous Ideas

#### Function Components

In my prototype, I allow functions as entries in the database.
Maybe useful for presenting a cohesive API?
Examples from my prototype model:
  + clock()
  + create_segment()
  + destroy_segment()

#### Grids of Entities

An Archetype could define a regular grid to place Entities on and provide tools
for efficiently working with grids of Entities. For example this could be used
to implement extracellular diffusion at a course granularity.

#### Spatial Partitioning Structures
 - For performing fast nearest neighbor searches.

#### Temporary Buffers

It might be nice to have working data be managed by the database, but not
persistently stored. The programmer would be responsible for free'ing it when
they're done with it.

#### Linear Systems
 - different algorithm than NEURON ...
 - exact but not accurate :(

#### Bookkeeping
 - valid/invalid?
 - run on change?
 - data logging buffers?

## Application Programming Interface
 - The database program and API will run within a single OS process, and may be
   further restricted to a single thread of execution.
 - The database will provide a python API.
 - The API is callable from a single thread of execution.

[tbd]
