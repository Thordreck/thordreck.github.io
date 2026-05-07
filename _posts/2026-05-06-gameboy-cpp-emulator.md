---
layout: post
title:  "Writing a gameboy emulator in C++23"
date:   2026-05-06 08:26:41 +0200
categories: programming c++
tags: c++ programming emulation
---

I have been working on an [original gameboy emulator](https://github.com/Thordreck/gameboy-cpp) for a couple months now. My main goal was to familiarize myself with all the new stuff that got added into C++ 20 and 23, try out new tools and document the process. This is not intended as a guide on how to write a gameboy emulator - the documentation online is extensive and of better quality than what I could produce myself. I am merely standing on the shoulders of giants.

Instead, I want to focus on what a project written from scratch in C++23 looks like, what currently works and what doesn't, the pitfalls I fell into and how I tackled some specific problems. While there are still lots of stuff missing, I think the project is at a stage where I can draw some conclusions. Before we delve into the details, though, here are some screenshots of the emulator running:


<div style="display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px;">
  <img src="/assets/images/gameboy-pokemon-red.png">
  <img src="/assets/images/gameboy-tetris.png">
  <img src="/assets/images/gameboy-super-mario-land.png">
  <img src="/assets/images/gameboy-legend-of-zelda.png">
</div>

# The original plan

I started this project with some goals in mind:

* Extensive use of templates and concepts.
* All code had to be structured in modules, instead of the traditional header + source file combo.
* No usage of macros.
* Favor composition over inheritance.
* Testing should not be painful.

I am proud to announce that I have failed miserably in pretty much all of these goals. I have learnt a bit along the way, so hopefully this could be useful to someone. 

I have also came across a couple of [CppCon talks by Tom Tesch](https://www.youtube.com/watch?v=4lliFwe5_yg) recently, where he also talks about modern C++ applied to gameboy emulation. While his approach is fundamentally different to mine (he tried to refactor and modernize an already existing codebase), I share his conclusions. Good code still remains good code, even if it was written in what's now considered "old" C++. Newest versions, however, offer more tools to write such good code.


# Concepts

I wholeheartedly think concepts are one of the best additions to the language I have seen in a while. They offer new syntax to define the requirements a type must satisfy to be a valid candidate for a templated function. Here's an example on how I used them to model the memory bus (note how simple concepts can be combined to define more complex ones):

{% highlight c++ %}

using memory_data_t = std::uint8_t;
using memory_address_t = std::uint16_t;

template<typename T>
concept ReadOnlyMemory = requires(
    const T& memory, 
    const memory_address_t address)
{
    { memory.read(address) } -> std::convertible_to<memory_data_t>;
};

template<typename T>
concept WriteOnlyMemory = requires(
    T& memory, 
    const memory_address_t address, 
    const memory_data_t value)
{
    { memory.write(address, value) } -> std::same_as<void>;
};

template<typename T>
concept Memory = WriteOnlyMemory<T> && ReadOnlyMemory<T>;

{% endhighlight c++ %}

It took me a while to get used to the syntax, as there are multiple ways to define concepts and to apply them to a function/type, but once it clicks there's absolutely no reason whatsoever to write templates without them. The basic idea is that the `requires` body must result in an expression that compiles. Here's an example on how I used it in a cpu instruction:


{% highlight c++ %}

struct or_a_hl
{
    static constexpr step_t num_steps(const cpu::cpu_state&) { return 2; }

    template<memory::ReadOnlyMemory Memory>
    static void execute(cpu::cpu_state& cpu, const step_t step, const Memory& memory)
    {
        if (step == 1)
        {
            cpu.reg.a() = cpu.reg.a() | memory.read(cpu.reg.hl());

            cpu.reg.flags().z = cpu.reg.a() == 0;
            cpu.reg.flags().n = false;
            cpu.reg.flags().h = false;
            cpu.reg.flags().c = false;
        }
    }
};

{% endhighlight c++ %}

As seen in the snippet above, the simplest way to use a concept is to use its name instead of `class`  or `typename` in the template parameter.
While everything that can be done with concepts could be done without them already, they do lower the entry barrier. First of all, they serve as documentation on what the function expects. More importantly, when you try to use a type that does not satisfy it, you get an actual error your average God-fearing developer with a 9-to-5 job can understand, instead of pages and pages of failed template instantiations.

By relying more on templates, I have noticed that I tend to write code that is easier to test. Take the cpu instruction from above, for example. If I wanted to test it, I could simply write a wrapper around an `std::array` that exposes a `read()` and `write()` methods and pass it, instead of using the real implementation. It also had the side-effect of forcing me to write classes in a way that rely on dependencies being passed around when needed, rather than classes creating them on their own and taking ownership or receiving them in their constructor. The former would only require a method to be a template, while the latter would force the class itself to be templated, which I tried to avoid to reduce complexity.

I think concepts are the final push to drop both the "template" and "meta" from template metaprogramming. Template metaprogramming is just normal programming.

# Modules

I think modules are also a great addition and a welcomed paradigm shift, but tooling support is not there yet. Do not, and I repeat, do not use modules in a serious long term project in its current state.

Let's get the basics first before diving into the issues. Modules offer a way to compartmentalize code into units. Inside a unit, constructs such as types, classes, functions and concepts can be exported selectively so they are visible to consumers that import them. Here's an example of the top-level module that implements the gameboy interruptions:

{% highlight c++ %}

export module interrupts;

export import :common;
export import :dispatch;
export import :enable;
export import :request;
export import :service;
export import :processor;

{% endhighlight c++ %}

The module is divided into partitions. Here's an excerpt of the first `:common` partition (note the use of the `export` keyword before each definition meant to be public):

{% highlight c++ %}

export module interrupts:common;

import std;
import memory;
import cpu;

namespace interrupts
{
    export constexpr std::uint16_t ie_address = 0xFFFF;
    export constexpr std::uint16_t if_address = 0xFF0F;

    export constexpr std::uint8_t if_mask = 0x1F;
    export constexpr std::uint8_t ie_mask = 0x1F;

    export template<typename T>
    concept InterruptDescriptor = requires(const T& descriptor)
    {
        { descriptor.ie_flag() } -> std::convertible_to<std::uint8_t>;
        { descriptor.if_flag() } -> std::convertible_to<std::uint8_t>;
        { descriptor.handler_address() } -> std::convertible_to<std::uint16_t>;
    };
}

{% endhighlight c++ %}

To use interruptions anywhere else, preface the code with `import interrupts;`. The interrupts module itself imports several modules as dependencies. Note the `import std;`, that replaces individual `include` directives for the standard library, such as `#include <cstint>`. In terms of actual file hierarchies, modules replace `.h` and `.cpp` files with `.cppm` ones, as shown in the picture below:

![File hierarchy of a C++ module](/assets/images/module-file-hierarchy-example.png)

Building code that use modules using CMake is more or less the same as usual. The only difference being that module files must be tagged as such and some additional target properties must be set:

{% highlight cmake %}

add_library(interrupts)
target_compile_features(interrupts PUBLIC cxx_std_23)

set_target_properties(interrupts PROPERTIES CXX_SCAN_FOR_MODULES ON CXX_MODULE_STD ON)
target_link_libraries(interrupts cpu memory utilities)

target_sources(interrupts PUBLIC 
	FILE_SET 
		CXX_MODULES 
	FILES 
		${CMAKE_CURRENT_SOURCE_DIR}/interrupts.cppm
		${CMAKE_CURRENT_SOURCE_DIR}/interrupts_common.cppm
		${CMAKE_CURRENT_SOURCE_DIR}/interrupts_request.cppm
		${CMAKE_CURRENT_SOURCE_DIR}/interrupts_enable.cppm
		${CMAKE_CURRENT_SOURCE_DIR}/interrupts_dispatch.cppm
		${CMAKE_CURRENT_SOURCE_DIR}/interrupts_service.cppm
		${CMAKE_CURRENT_SOURCE_DIR}/interrupts_processor.cppm
)

{% endhighlight cmake %}

Modules have helped me to reason differently about how I structured the whole project. With traditional header files, I used to split code in terms of classes and implementations. With modules, however, I started grouping code based on their intent and functionality. Note how the files in the `interrupts` module shown above have more general names describing their intent, rather than their actual classes/functions. It's also nice to be able to write code directly into the declaration instead of having to jump around header and cpp files.

And now, for the ugly bits. I initially intended to write this project using Visual Studio Code. It's what I have been using recently and I saw no reason to change. Unfortunately, VS Code's Intellisense does not work properly with modules. While the project compiles, it's basically impossible to work with it, since it does not recognize `import` nor `export`, code completion does not work, etc. I switched to Visual Studio, as I read modules support was more mature there. And it kind of worked, until it didn't. It recognized module syntax properly, but it would randomly choke on some modules while recognizing and being able to go to the definitions of others. Turns out the IDE relies on the existence of IFC files for things like code completion/navigation, etc and, for some reason, it would sometimes not generate them. I tried to forcefully generate them with some CMake magic but it was not reliable. 

I ended up switching to CLion. Both the 2025 and 2026 version seem to work fine out of the box. While I do not have anything against it, it is unfortunate that there are currently no other viable options.

It's finally time to address the elephant in the room when it comes to modules: third party dependencies. External dependencies should not be a problem in theory, since header files can be included in modules just fine. While that is certainly an option, I wanted to try implementing everything using modules, so I tried to wrap all third party libraries in my own modules. See the example below, that adapts the initialization process of the SDL library in a module (ignore any possible issues due to copies, moves, default constructor, etc):

{% highlight c++ %}

module;
#include <SDL3/SDL_init.h>

export module sdl:init;

import :common;
import std;

namespace sdl
{
    export enum class init_flags : std::uint32_t
    {
        audio = SDL_INIT_AUDIO,
        video = SDL_INIT_VIDEO,
        joystick = SDL_INIT_JOYSTICK,
        haptic = SDL_INIT_HAPTIC,
        gamepad = SDL_INIT_GAMEPAD,
        events = SDL_INIT_EVENTS,
        sensor = SDL_INIT_SENSOR,
        camera = SDL_INIT_CAMERA,
    };

    export class session
    {
    public:
        static result<session> create(const init_flags flags)
        {
            if (SDL_Init(std::to_underlying(flags)))
            {
                return session();
            }

            return std::unexpected(SDL_GetError());
        }

        ~session() { SDL_Quit(); }
    };
}

{% endhighlight c++ %}


Adapting external libraries as modules is just a matter of identifying the individual header files and including them in a top `module` section. This approach works perfectly well for code that does not rely heavily on macros, since those cannot be exported. 

Because of this, my first taste of defeat came when integrating [Tracy](https://github.com/wolfpld/tracy) into the project. Tracy is a profiler library that relies on macros like `ZoneScoped` extensively for code instrumentation. I tried to replace them with my own non-macro functions but performance took a hit, since it uses some macro shenanigans to define stuff at compile time. I also had to trust the compiler was smart enough to remove my empty implementations on non-profiling builds, something you don't have to worry about when using macros.

The final nail in the coffin for me was Qt. While I had a working interface implemented using [imgui](https://github.com/ocornut/imgui) and [sdl3](https://github.com/libsdl-org/SDL), I wanted to support additional frontends. It turned out to be a nightmare. Forget about wrapping it in a module, it was not even possible to use it directly. This was not only due to their reliance on your code inheriting from their own types like `QObject` and macros like `Q_OBJECT`, `Q_PROPERTY` and `QML_ELEMENT`, but due to their code generation system. Trying to define a QObject class within a module resulted in compilation errors, since Qt's code generation system does not support them. The only workaround I found was to fallback to header and cpp files just for types interacting directly with Qt, compile them separately and link those against my own modules.

All in all, I think modules are the future, but we are still years away from them being usable in a serious project.

# Performance

The original gameboy is a platform easy to emulate with the hardware available nowadays, but it's still important to write the code for it in a sensible way and to have good tools at your disposal to measure performance. This section is not a showcase of C++23's capabilities per se, but rather a collection of small things I have learnt along the way.

The first low hanging fruit to improve performance is to enable link-time optimization (LTO). It scans your whole project and it might find more places where functions can be inlined. It has the downside of increasing compilation times, but this project was small enough that I did not notice any difference. It did, however, provide a significant boost in performance. In CMake it can be enabled as follows:

{% highlight cmake %}

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RelWithDebInfo ON)

{% endhighlight cmake %}

I have found that writing code that can be inlined to be extremely important for performance. This, however, increases complexity, as commonly used type erasure mechanisms like function pointers or `std::function` make it nearly impossible. Templates and concepts are a good way to write code that can be inlined.

As previously mentioned, I have used Tracy as the main profiler. It's an useful tool to have an overview of where your code spends most of the time, as shown in the screenshot below:

![Tracy profiling example](/assets/images/gameboy-tracy-profiler-example.png)


Looking at the right side window in the profiler, it's possible to see that a huge chunk of the time is spent reading from and writing to the memory bus or ticking the different components. This kind of profiling has helped me realize when, for example, the simple act of invoking a function was starting to become a bottleneck. My initial implementation would simply call step to each component (cpu, timers, graphics) in each clock step. With the main clock running at 4Mhz, that's a lot of function calls over a frame. Instead, the usual approach in this scenario is to instruct the emulator to advance a number of ticks at once, thus reducing the overhead of invoking the function each time.

Sometimes this kind of high-level profiling is not enough to decide where to spend efforts optimizing. In those cases it's necessary to switch to vendor-specific profiling tools that can provide more low-level info. In my current setup I have used AMD uProf. The tool is capable of reporting the most prominent hotspots and give you the assembly that correspond to a piece of code. Here's an screenshot:


![AMD uProf example](/assets/images/gameboy-amd-uprof-example.png)

This has proved to be invaluable to not only improve performance, but to also gain a better instinct on what's more performant in general. Here's some of the things this has helped me catching:

* I had branches in some parts of the memory bus to discard regions of memory that were unused earlier. It turned out to be better to simply unify all cases into a huge switch block that compiled to a jump table.
* I made the assumption that some template code that used `std::tuple` was compiling into a jump table. This was wrong. I ended up having to switch to a macro instead of a template.

# Testing

For testing I have opted to use [doctest](https://github.com/doctest/doctest) instead of other frameworks like [googletest](https://github.com/google/googletest). Not only is easier to integrate, since it's a header-only file, but I think the larger scope of google test and its mocking capabilities end up being detrimental more often than not. 

Writing tests for an emulator is somehow different to what I'm accustomed to. At least in my case, I did write unit tests for things like cpu instructions or interrupts at first, but I quickly switched to well-known test roms. I treat these as a sort of end-to-end tests, since they usually rely on several things working like the screen or timers, even though they might be testing only one specific functionality. Automating these comes on a case-by-base basis, as each rom report their results in different ways.

There's the infamous acid2 test, originally conceived for web browsers, that has been ported as a gameboy rom and made publicly available. Below there's an screenshot of its result:


![Acid2 test rom result](/assets/images/gameboy-acid-test-result.png)

This is the kind of test that can be easily automated by comparing the emulator's framebuffer and a ground truth image.

Other tests do not rely on the emulator having a functional screen, and instead send their results through the serial port. And then there are those that are a combination of the two, where some magic number is sent through the serial port to indicate a test has failed, but details on why are printed on the screen. This is the case with some test suites like [mooneye](https://github.com/Gekkio/mooneye-test-suite). 

The vast majority of test roms I have found end up using the screen to display their results. While this is not an issue, I wish it was easier to somehow attach and display images as test results using CTest.

# What's next?

There are still several bugs, mainly related to graphics, that I have to fix. My goal is to pass all [mooneye](https://github.com/Gekkio/mooneye-test-suite) tests. Sound and save files/states are also missing. Some parts of the codebase would benefit from a refactor as well, mainly the graphics and interrupt modules.

I would also like to keep measuring performance and see where I can improve it. I recently came across a [CppCon talk](https://www.youtube.com/watch?v=g_X5g3xw43Q) that discussed how to write more cache-friendly code, with tips like re-ordering members to improve alignment and so on. 

Adding better testing would be nice too. Having some kind of automated performance tests that can catch drops in performance could be extremely useful. I think the [nanobench](https://github.com/martinus/nanobench) library might be the right tool for this.

I would also love to try porting the emulator to PS Vita. I have had one lying around unused for years, so I might as well dust it off and use it for something. I don't have high hopes for it, though, since the codebase uses modules and other modern features. But you never know. 