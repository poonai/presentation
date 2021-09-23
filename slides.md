---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
---

# How to build debuggers in Go

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---

# About me


I'm Poonai, your friendly meta cat. I'm one of the co-founder👨‍💼 of [Streak](https://streakcard.com) and taking care of technologies and tech team.

I've experience in storage, distributed systems and networking. In my free time, I build side projects. Currently, I'm building [Quicklog](https://quicklog.dev)🔭  at my free time.

If you want to reach out, you can DM me on [Twitter](https://twitter.com/poonai_)(@poonai_)🐦.  

<br>
<br>

<!--
You can have `style` tag in markdown to override the style for the current page.
Learn more: https://sli.dev/guide/syntax#embedded-styles
-->

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent; 
  -moz-text-fill-color: transparent;
}
</style>

---

# How do you debug programs🤔?

1) **fmt.Println(`🤷‍♀️`)**
2)  **Delve**


---
layout: image-right
image: https://source.unsplash.com/collection/94734566/1920x1080
---

# Demo with Delve
---

# Common Functionality

1) Breakpoint
2) Step in 
3) Step out
4) Inspect Variables


---

# What do we have?
<br>

- Binary file which we want to debug.
- source code

---

# DWARF comes to rescue

 DWARF is a debugging file format used by many compilers and debuggers to support source level debugging. It addresses the requirements of a number of procedural languages, such as C, C++, and Fortran, and is designed to be extensible to other languages. DWARF is architecture independent and applicable to any processor or operating system. It is widely used on Unix, Linux and other operating systems, as well as in stand-alone environments. source: http://www.dwarfstd.org/


<br>
<br>
- Mapping of programing language to machine code

---

# Mapping the binary to source code.

```
objdump --dwarf=decodedline ./sample
```

```
File name                            Line number    Starting address    View    Stmt

/home/poonai/debugger-example/sample.go:
sample.go                                      9            0x498200               x
sample.go                                      9            0x498213               x
sample.go                                     10            0x498221               x
sample.go                                     11            0x498223               x
sample.go                                     11            0x498225        
sample.go                                     12            0x498233               x
sample.go                                     12            0x498236        
sample.go                                     13            0x4982be               x
sample.go                                     13            0x4982cb        
sample.go                                     11            0x4982cd               x
sample.go                                     12            0x4982d2        
sample.go                                      9            0x4982d9               x
sample.go                                      9            0x4982de        
sample.go                                      9            0x4982e0               x
sample.go                                      9            0x4982e5    
```

--- 

# Trick to stop the program

- CPU stop the execution when it's find `0xcc`

--- 

# Shall we rewrite the binary with the trap code?
<br>
<br>
<div v-click><center><h1>Nahh</h1></center></div>

--- 

# What do we do then?
<br>
<br>
<div v-click><center><h1>Ptrace</h1></center></div>
<div v-click><center><p>ptrace is a system call found in Unix and several Unix-like operating systems. By using ptrace (the name is an abbreviation of “process trace”) one process can control another, enabling the controller to inspect and manipulate the internal state of its target. ptrace is used by debuggers and other code-analysis tools, mostly as aids to software development. source:wikipedia</p></center></div>

---

# Let's start the program

```go
process := exec.Command("./sample")
process.SysProcAttr = &syscall.SysProcAttr{
  Ptrace: true,
  Setpgid: true,    
  Foreground: false,
}
process.Stdout = os.Stdout
if err := process.Start(); err != nil {
    panic(err)
}
```

--- 

# Insert the trap code.

```go 
unix.PtracePokeData(pid, addr, []byte{0xCC})
```

```go
func setBreakpoint(pid int, addr uintptr) []byte {
    data := make([]byte, 1)
    if _, err := unix.PtracePeekData(pid, addr, data); err != nil {
        panic(err)
    }
    if _, err := unix.PtracePokeData(pid, addr, []byte{0xCC}); err != nil {
        panic(err)
    }
    return data
}
```


--- 

# Run and wait for the code to pause

```go
if err := unix.PtraceCont(pid, 0); err != nil {
     panic(err.Error())
 }
 // wait for the interupt to come.
 var status unix.WaitStatus
 if _, err := unix.Wait4(pid, &status, 0, nil); err != nil {
     panic(err.Error())
 }
 fmt.Println("breakpoint hit")
```

--- 

# How do we resume the program?

<br>
<br>
<div v-click><center><h1>Revert back to original data</h1></center></div>

--- 

# Intro to registers

![registers](https://raw.githubusercontent.com/poonai/poonai.github.io/deploy/static/img/registersintro.png)

--- 

# Register manipulation using ptrace
```go
    regs := &unix.PtraceRegs{}
    if err := unix.PtraceGetRegs(pid, regs); err != nil {
        panic(err)
    }
    regs.Rip = uint64(addr)

    if err := unix.PtraceSetRegs(pid, regs); err != nil {
        panic(err)
    }
```

---

# Reset the breakpoint

```go
func resetBreakpoint(pid int, addr uintptr, originaldata []byte) {
   // revert back to original data
    if _, err := unix.PtracePokeData(pid, addr, originaldata); err != nil {
        panic(err.Error())
    }
    // set the instruction pointer to execute the instruction again
    regs := &unix.PtraceRegs{}
    if err := unix.PtraceGetRegs(pid, regs); err != nil {
        panic(err)
    }
    regs.Rip = uint64(addr)

    if err := unix.PtraceSetRegs(pid, regs); err != nil {
        panic(err)
    }
    if err := unix.PtraceSingleStep(pid); err != nil {
        panic(err)
    }
    // wait for it's execution and set the breakpoint again
    var status unix.WaitStatus
    if _, err := unix.Wait4(pid, &status, 0, nil); err != nil {
        panic(err.Error())
    }
    setBreakpoint(pid, addr)
}
```

--- 
<br>
<br>
<br>
<br>
<br>
<br>
<center><h1>Demo Time</h1></center>

---
# Quciklog demo

---
layout: two-cols
---

# Thank you
<iframe src="https://giphy.com/embed/zwDNti5vWFujS" width="360" height="268" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/naruto-shippuden-kakashi-hatake-zwDNti5vWFujS"></a></p>
::right::

# Contact
<br>
Twitter: @poonai_

Quicklog: https://quicklog.dev

--- 
# Plug

## Hiring Backend Interns for Streak


