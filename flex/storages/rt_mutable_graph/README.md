# Real-Time Mutable Graph

## 1. Introduction

`rt_mutable_graph` provides a real-time mutable graph store. It supports MVCC and is optimized for executing read and insert transactions concurrently.

## 2. Schema and Configuration

`rt_mutable_graph` adopts property graph model:

- Vertices are the entities in the graph, representing objects or concepts.
    - Each vertex has a label that describes the type of entity it represents, and a set of properties that describe its characteristcs.
    - Vertices of each type share same property schema.
- Edges are the relationships between vertices, representing the associations or interactions between entities.
    - Each edge has a label that describes the type of interaction, and a set of properties that describe its characteristcs.
    - Edges with same label-triplet(source vertex label, edge label, destination vertex label) share property schema.

In `rt_mutable_graph`, schema is defined [here](./schema.h). To initialize an `MutablePropertyFragment`, an `Schema` object should be provided. There are 2 ways to setup the `Schema` object:

- Invoking the member functions of `Schema`, such as `add_vertex_label`, `add_edge_label`, etc.
- Parsing an `Schema` object from a YAML configuration file, using `Schema::LoadFromYaml`.

The configuration file ([modern graph example](./modern_graph/modern_graph.yaml)) defines the graph schema and the raw files of each type of vertices and edges. Also, a set of stored procedures can be registered here. With this configuration file, a `MutablePropertyGraph` object can be initialized.

Here is an example of a configuration file:

```yaml
name: modern
store_type: mutable_csr
stored_procedures:
  directory: plugins
  enable_lists:
    - libxxx.so
schema:
  vertex_types:
    - type_name: person
      x_csr_params:
        max_vertex_num: 100
      properties:
        - property_id: 0
          property_name: id
          property_type:
            primitive_type: DT_SIGNED_INT64
        - property_id: 1
          property_name: name
          property_type:
            primitive_type: DT_STRING
        - property_id: 2
          property_name: age
          property_type:
            primitive_type: DT_SIGNED_INT32
      primary_keys:
        - id
    - type_name: software
      x_csr_params:
        max_vertex_num: 100
      properties:
        - property_id: 0
          property_name: id
          property_type:
            primitive_type: DT_SIGNED_INT64
          x_csr_params:
        - property_id: 1
          property_name: name
          property_type:
            primitive_type: DT_STRING
        - property_id: 2
          property_name: lang
          property_type:
            primitive_type: DT_STRING
      primary_keys:
        - id
  edge_types:
    - type_name: knows
      vertex_type_pair_relations:
        source_vertex: person
        destination_vertex: person
        relation: MANY_TO_MANY
        x_csr_params:
          incoming_edge_strategy: Multiple
          outgoing_edge_strategy: Multiple
      properties:
        - property_id: 0
          property_name: weight
          property_type:
            primitive_type: DT_DOUBLE
    - type_name: created
      vertex_type_pair_relations:
        source_vertex: person
        destination_vertex: software
        relation: ONE_TO_MANY
        x_csr_params: 
          incoming_edge_strategy: Multiple
          outgoing_edge_strategy: Single
      properties:
        - property_id: 0
          property_name: weight
          property_type:
            primitive_type: DT_DOUBLE
```

Notes:

- Currently we only support one primary key, and the type has to be `DT_SIGNED_INT64`.
- All implementation related configuration are put under x_csr_params.
  - `max_vertex_num` limit the number of vertices of this type:
    - The limit number is used to `mmap` memory, so it only takes virtual memory before vertices are actually inserted.
    - If `max_vertex_num` is not set, a default large number (e.g.: 2^48) will be used.
  - `incoming/outgoing_edge_strategy` specifies the storing strategy of the incoming or outgoing edges of this type, there are 3 kinds of strategies
    - None: no edge will be stored
    - Single: only one edge will be stored
    - Multiple(default): multiple edges will be stored

## 3. Vertex Management

### 3.1 ID mapping

In `rt_mutable_graph`, each vertex is assigned an internal ID starting from zero, which is a consecutive integer. This ID can uniquely identify a vertex and can also be used as an offset to quickly access data related to the vertex in property tables and adjacency lists. 

The mapping between the external and internal IDs is maintained in [LFIndexer](../../utils/id_indexer.h), which is implemented based on ska::flat_hash_map and supports concurrent, lock-free insertion of elements.

### 3.2 Vertex properties storage

The properties of vertices are maintained in [Table](../../utils/property/table.h) in a column-oriented manner. Storing data in a column-oriented manner makes it more compact, and users can access only the necessary columns as per their requirements, without introducing redundant memory/disk access.

All the columns are implemented with `mmap`-ed memory. The `max_vertex_num` field of Schema will be used to ensure virtual memory region is large enough.

### 3.3 Vertex insert

When inserting a new vertex, a self-incremented internal ID (generated by an atomic integer) will be assigned to it. The properties of the vertex will be inserted into the property table. The internal ID will be inserted into the LFIndexer.

## 4. Edge Management

 Only insertion is allowed now. Each edge will be assigned with a timestamp to specify since when it is visable.

### 4.1 Memory layout

A csr-like data structure [`MutableCsr`](./mutable_csr.h) is used to store the edges of each direction of each label-triplet.

There are two components of `MutableCsr`:

- `MutableCsr::adj_lists_`: a list of `MutableAdjList`, each `MutableAdjList` stores the edges of a vertex
- `MutableCsr::locks_`: a list of `SpinLock`, each `SpinLock` is used to protect the corresponding `MutableAdjList` from concurrent modifications

`MutableAdjList` has 3 members:

- `nbr_t* buffer_`: a pointer to the memory region where the edges are stored, the memory is allocated and will be de-allocated by `ArenaAllocator`s
- `std::atomic<int> size_`: the number of edges stored in the `buffer_`
- `int capacity_`: the capacity of the `buffer_`

### 4.2. Edge insert and versioning

When inserting an edge (src, dst) to `MutableCsr csr` as an outgoing edge:

```
csr.lock_[src].lock() // take the lock first
if csr.adj_lists_[src].capacity_ == csr.adj_lists_[src].size_: // need resizing
    allocating a double-sized new_buffer;
    copy the edges from csr.adj_lists_[src].buffer_ to new_buffer;
    assign new_buffer to csr.adj_lists_[src].buffer_;
csr.adj_lists_[src].buffer_[csr.adj_lists_[src].size_.fetch_add(1)] = dst; // append the edge to the adj list
csr.lock_[src].unlock()
```

With the protection of the `SpinLock`s, the edge insertion is thread-safe. The synchronization and isolation between read-threads and write-thread are achieved by:

- Timestamps on edges: edges are only visible to read-threads if their timestamps are smaller than the timestamp of the read-threads.
- The lifecycle of all adjacency lists are managed by `ArenaAllocater`s, which means resizing the adjacency list of a vertex won't de-allocate the old adjacency list immediately if there are threads reading on it.


### 4.3 Edge data storage

Unlike the property table of vertices, the property table of edges is not column-oriented. The reason is that the number of properties of edges is usually small, and the number of edges is usually large. So the property table of edges is implemented as a row-oriented table.

Fow now, only one property is supported for edges, but developers can define a struct with multiple fields to store multiple properties.


## 5. Stored procedures

Stored procedures are shared libraries that can be loaded by `rt_mutable_graph` to extend its functionality. The stored procedures are loaded by `dlopen`.


## 6. Durability

After loading and constructing a graph from input files for the first time, the graph will be dumped as data files in a directory. The directory will be used as the input directory for the next time the graph is loaded. 

The dumped data files are immutable. All modifications (with InsertTransactions and UpdateTransactions) will be encoded as write ahead log.
