## Creating a Dynamically Linked Functions Library in CPP

Proposers

* Soumya Duriseti
* Tim Meehan

## [Related Issues]

https://github.com/facebookincubator/velox/pull/1005

## Summary

This proposed change expands the dynamic function loading ability to cpp user defined functions (UDFs). The Prestissimo worker is to access said code. The dynamic functions are to be loaded upon running an instance of the presto server. In the presto server instance, it will search for any .so or .dylib files and load them using this library.
## Background

Currently, on Presto, any Java UDFs can be loaded dynamically. This is an important feature for any client who wants their Presto build to be lightweight and to protect any non-public code or information from having to be in the open source space. Creating UDFs is an important functionality Presto has and its competitors offer as well. Being able to load CPP functions dynamically will extend the offering in Prestissimo.

### [Optional] Goals
Should be able to register all functions on runtime as theyre specified in the plugin directory. 
### [Optional] Non-goals
Security concerns: There are some security concerns associated with using the built in plugins. Using the dlopen library, we run the risk of opening unsecure unknown shared objects especially given the lack of any form of validation. On CPP, we share the same limitations as the functionality in Java with a noted exception of Java Presto running in a VM environmnet while the CPP version will be run locally. We will not be addressing these security concerns through code at this time.

## Proposed Implementation
The user can register their functions dynamically by calling loadDynamicLibraryFunctions() with the path to their shared library (files ending in .so in linux or .dylib in MacOS). At the time of running an instance of the PresterServer, if any shared library files exist in plugin directory, they get loaded on start up. Alternatively, the user can call loadDynamicLibraryFunctions() elsewhere and specify the exact location of these files and load them upon execution of this code.

For dynamically loaded function registration, the format followed is mirrored of built-in function registration with some noted differences. For instance, the below example function uses the extern "C" keyword to protect against name mangling. additionally, a registry() function call is also necessary here.

namespace facebook::presto::functions {

template <typename TExecParams>
struct Dynamic123Function {
  FOLLY_ALWAYS_INLINE bool call(int64_t& result) {
    result = 123;
    return true;
  }
};

} // namespace facebook::presto::functions

extern "C" {

void registry() {
  facebook::velox::registerFunction<
      facebook::presto::functions::Dynamic123Function,
      int64_t>({"dynamic_123"});
}
}

The general process is as follows:

1. What modules are involved
    dlfcn.h (Dynamic linking library)
    udf.h (for function registerations)
2. Any new terminologies/concepts/SQL language additions
    None.
3. Method/class/interface contracts which you deem fit for implementation.
    None.
4. Code flow using bullet points or pseudo code as applicable
    1. User is to create a shared library (in the form of a .so or .dylib file) for the UDF they wish to register. This is freeform with the noted exception that they follow the velox function registry API to create their UDF. This will be noted in the documentation. One way to create shared libraries is through CMake using the SHARED keyword in the CMakeLists.txt.
    2. User will proceed to place their .so/.dylib files into the plugin directory.
    3. Upon running the PrestoServer, we will scan the plugin directory to load the .so/.dylib files dynamically using a call to loadDynamicLibraryFunctions. This function uses dlopen() to dynamically load these files.
5. Any new user facing metrics that can be shown on CLI or UI.
    None.
## [Optional] Metrics

We indend to use Pbench for performing performance testing. This library's effectiveness can be measured by successful completion of registering all of UDFs and validating their proper registration using the CLI with a call to SHOW FUNCTIONS and by successful completion of the process.

## [Optional] Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
no impact as this is a new offering.
- If we are changing behaviour how will we phase out the older behaviour?
- If we need special migration tools, describe them here.
- When will we remove the existing behaviour, if applicable.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
I will be including a README with these changes to explain to users how to properly use the dyllib functionality.
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
None.

## Test Plan

Will be writing an E2E test which will go through the entire process and validate the function registering with a SHOW FUNCTIONS call. We will also use Pbench to do performance testing.