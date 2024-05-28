
# R and D

This is the result of various experimentations exploring the security properties
of the software written in C and C++ using LLVM IR. 

Main theme here involves 

- creating custom graphs that handles function calls (and virtual method calls)
  with inter procedural control-flow and data-flow analysis.

- callsite sensitivity: currently, 6. [depending on available RAM].

- Flow-insensitive and Context-insensitive analysis: utilizing Andersen's subset
  based unification analysis.

- Utilizing the information obtained from above analysis through a LLVM
  middle-end pass to construct the required graphs.

- Use SQLAlchemy / any other alternative to conviniently form the queries using
  the Mathematics mentioned in this repo.  
