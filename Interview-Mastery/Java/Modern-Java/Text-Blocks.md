# Text Blocks

- Purpose: represent multi-line string literals without escape-sequence mess
- Delimiter: opening """ must be followed by a line terminator; closing """ determines indentation
- Leading whitespace stripped according to the position of the closing """
- Indent algorithm: common leading whitespace across all lines is removed
- Trailing whitespace on each line is stripped
- Escape sequences still work inside text blocks
- New escape sequences: \s forces a trailing space, \linebreak suppresses the line break for line continuation
- Java does not have native string interpolation; use String.formatted() or formatted() for variable substitution
- SQL example: embed multi-line queries without concatenation or + signs
- JSON example: write JSON literals directly without escaping every quote
- IDE folding support: most IDEs collapse text blocks into a single line in the editor
