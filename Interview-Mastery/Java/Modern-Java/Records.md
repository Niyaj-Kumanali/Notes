# Records

- Purpose: transparent data carriers that replace boilerplate-heavy POJOs
- Declaration syntax: record Point(int x, int y) {}
- Canonical constructor matches the component list exactly
- Compact constructor omits parameter list, allows implicit field assignment with validation logic
- Custom constructor must delegate to canonical via this()
- Accessor methods match component names (Point.x(), Point.y()) not JavaBean getX()/getY()
- equals() compares all components structurally
- hashCode() derived from all components
- toString() returns a concise string of all components
- Cannot extend any class (records implicitly extend java.lang.Record)
- Cannot declare instance fields beyond the component list
- Can implement interfaces and define static fields/methods
- Local records: records defined inside a method for intermediate data grouping
- Records work well in collections as keys due to proper equals/hashCode
- JPA/Hibernate require a no-arg constructor workaround via lombok or a custom constructor with default values
