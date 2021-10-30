# Shaders on the Wii U

## Introduction
So why am I making this "guide"? I recently ported GTA 3 to the Wii U and with it all the shaders of renderware. This is pretty undocumented territory so I hope this guide will be useful in the future.  
The Wii U's graphics library is called GX2 and it has it's similarities with OpenGL.  
The only issues are shaders. You can't simply compile GLSL shaders at runtime. Instead shaders are pre-compiled and only read at runtime.  
Before reading through this, I expect you to know a bit about shaders in general and GLSL.

## Getting started
In this tutorial we'll write our shaders in assembly and assemble them using latte-assembler into a format called GSH (GX2 Shader). This format can be read by GX2.  
Latte-assembler is part of decaf-emu and can be downloaded from the artifacts of the [latest workflow](https://github.com/decaf-emu/decaf-emu/actions) (You need a GitHub account to download artifacts).

## Writing our first shader
To assemble a shader we need a vertex and pixel shader.  
For that I'm going to create 2 files: `shader.vsh` for the vertex shader and `shader.psh` for the pixel/fragment shader.  
As a first shader we're simply going to write a shader that takes a position and color as the attributes and outputs it.    
Latte-assembler will parse comments for various options that can be set in the shader. Comments are prefixed with a semicolon (`; this is a comment`).  

### Vertex shader

This is how the vertex shader will look in the end:  
```
; $MODE = "UniformRegister"

; $ATTRIB_VARS[0].name = "position"
; $ATTRIB_VARS[0].type = "vec4"
; $ATTRIB_VARS[0].location = 0
; $ATTRIB_VARS[1].name = "color"
; $ATTRIB_VARS[1].type = "vec4"
; $ATTRIB_VARS[1].location = 1

; $SPI_VS_OUT_CONFIG.VS_EXPORT_COUNT = 0
; $NUM_SPI_VS_OUT_ID = 1
; $SPI_VS_OUT_ID[0].SEMANTIC_0 = 0

00 CALL_FS NO_BARRIER
01 EXP_DONE: POS0, R1
02 EXP_DONE: PARAM0, R2 NO_BARRIER
END_OF_PROGRAM
```

A GLSL equivalent would look something like this:
```glsl
in vec4 in_position;
in vec4 in_color;

out vec4 out_color;

void main() {
	gl_Position = in_position;
	out_color = in_color;
}
```

Let's break up the shader and take a more detailed look at it.  
  
```
; $MODE = "UniformRegister"
```
This sets the uniform mode of the shader. In this shader we don't use any uniforms so this doesn't really matter.  

```
; $ATTRIB_VARS[0].name = "position"
; $ATTRIB_VARS[0].type = "vec4"
; $ATTRIB_VARS[0].location = 0
; $ATTRIB_VARS[1].name = "color"
; $ATTRIB_VARS[1].type = "vec4"
; $ATTRIB_VARS[1].location = 1
```
These are the attributes. Attributes are the inputs from the CPU side (the values in the attribute buffers). Each attribute is specified in the `ATTRIB_VARS` array with an index, in case of color, it is 0.  

The `name` is the name of the attribute. It's required to initialize and refer to the attribute from the CPU side.  

The `type` is the variable type of the attribute. In this case a `vec4`, which is a vector that consists of 4 32-bit floats.  
For the position it stores the x, y, z and w coordinates, for color the r, g, b and a components.

The `location` refers to the location in the attribute buffer and will also affect the register this attribute will be stored in. Attributes in location 0 will be placed into register 1 (R1), attributes in location 1 into register 2 (R2), etc...  

The next few lines are to set up semantics. Semantics are the values passed from the vertex shader to the pixel shader.
```
; $SPI_VS_OUT_CONFIG.VS_EXPORT_COUNT = 0
```
This is the amount of semantics exported minus one. In this example we use one semantic (color), so the value is 0.
```
; $NUM_SPI_VS_OUT_ID = 1
```
This is the amount of `SPI_VS_OUT_ID`s we use. Every `SPI_VS_OUT_ID` can store 4 semantics.

```
; $SPI_VS_OUT_ID[0].SEMANTIC_0 = 0
```
This will initialize semantic 0. Each semantic used should be initialized to it's index. If there would be another semantic it should be set up like this `$SPI_VS_OUT_ID[0].SEMANTIC_1 = 1`.

```
00 CALL_FS NO_BARRIER
```
This will now start our actual shader program.

```
01 EXP_DONE: POS0, R1
02 EXP_DONE: PARAM0, R2 NO_BARRIER
```
We now output R1 into the vertex position, and pass R2 to PARAM0 of the pixel shader.

```
END_OF_PROGRAM
```
This ends the shader program.

### Pixel shader
This is how the pixel shader will look in the end:
```
; $MODE = "UniformRegister"

; $NUM_SPI_PS_INPUT_CNTL = 1

; $SPI_PS_INPUT_CNTL[0].SEMANTIC = 0
; $SPI_PS_INPUT_CNTL[0].DEFAULT_VAL = 1

00 EXP_DONE: PIX0, R0
END_OF_PROGRAM
```

A GLSL equivalent would look something like this:
```glsl
in vec4 in_color;

out vec4 out_color;

void main() {
	out_color = in_color;
}
```

Let's go through the pixel shader now.

```
; $MODE = "UniformRegister"
```
This will set the uniform mode, like in the vertex shader above.

```
; $NUM_SPI_PS_INPUT_CNTL = 1
```
This sets the number of inputs, we output from our vertex shader. Since we only input color, we set it to 1.

```
; $SPI_PS_INPUT_CNTL[0].SEMANTIC = 0
; $SPI_PS_INPUT_CNTL[0].DEFAULT_VAL = 1
```
This describes the inputs.  
`SEMANTIC` being the semantic index we set in the vertex shader.  
`DEFAULT_VAL` is the default value in case the semantic is not set. 1 would be a default of `0.0f, 0.0f, 0.0f, 1.0f`.  
The values of semantic 0 stored into R0, the one of semantic 1 into R1, ...

```
00 EXP_DONE: PIX0, R0
```
This will output R0 into the pixel color.

```
END_OF_PROGRAM
```
Like in the vertex shader above, this will end the shader program.

### Assembling the shader
To assemble the shader with latte-assembler we now run the following command:
```
./latte-assembler assemble --vsh=shader.vsh --psh=shader.psh shader.gsh
```
If it succeeded we now have a `shader.gsh`, which we can use in our GX2 applications.

## References
- http://developer.amd.com/wordpress/media/2013/10/evergreen_3D_registers_v2.pdf
- https://www.techpowerup.com/gpu-specs/docs/ati-r600-isa.pdf
- https://developer.amd.com/wordpress/media/2012/10/R600-R700-Evergreen_Assembly_Language_Format.pdf
- https://developer.amd.com/wordpress/media/2012/10/R700-Family_Instruction_Set_Architecture.pdf

## Some examples
- https://github.com/GaryOderNichts/librw/tree/gx2/src/gx2/shaders/shader_source
- https://github.com/yawut/SDL/tree/wiiu-2.0.9/src/video/wiiu/shaders
