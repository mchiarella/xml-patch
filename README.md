# xml-patch
Java implementation of RFC 5261: An XML Patch Operations Framework Utilizing XPath Selectors

With Gradle and Ant integration.

## Introduction

So you're interested in patching some XML documents? Here's a quick illustration of why RFC 5261 XML patches
are the way to go.

Original document:

    <example>
        <patchTool url="https://en.wikipedia.org/wiki/Regular_expression"/>
        <greeting>Hello sorrow</greeting>
    </example>


RFC 5261 patch:

    <diff>
        <replace sel="example/patchTool/@url">https://tools.ietf.org/html/rfc5261</replace>
        <add pos="before" sel="example/greeting"><valediction>Goodbye sorrow</valediction>
        </add>
        <replace sel="example/greeting/text()">Hello XML patch</replace>
    </diff>


Result of applying the patch to the original document:

    <example>
        <patchTool url="https://tools.ietf.org/html/rfc5261" />
        <valediction>Goodbye sorrow</valediction>
        <greeting>Hello XML patch</greeting>
    </example>


Check out [RFC 5261](https://tools.ietf.org/html/rfc5261) for more examples
and a detailed explanation of the patch language.


## Multi-Patch

RFC 5261 is great for patching one document at a time. If you need to patch many documents,
it may be convienient to bundle related patches into a 'multi-patch' document that can target
multiple, specific files.


Original file 'pets.xml':

    <pets>
        <pet type="dog" name="Frankie"/>
        <pet type="cat" name="Lucy"/>
    </pets>


Original file 'stores.xml':

    <stores>
        <store name="Pets R Us"/>
    </stores>


Multi-patch:

    <diffs>
        <diff file="pets.xml">
            <remove sel="//pet[@name='Lucy']"/>
        </diff>

        <diff file="stores.xml">
            <add sel="stores">
                <store name="Paw Pals"/>
            </add>
        </diff>
    </diffs>


As you can see, a multi-patch bundles zero or more RFC 5261 'diff' elements into a single document.
Each 'diff' element has a 'file' attribute specifying a relative path to the file targeted by the patch.

If your project configuration is spread across multiple XML files (web.xml, log4j.xml, foo.xml, etc.)
then a multi-patch is a convenient way to patch all of these files at once.


### Multi-Patch Composition

Multi-patches may include other multi-patches. This is useful if your project has multiple configurations
that share some common patches.


Multipatch file 'common-multi-patch.xml':

    <diffs>
        <diff file="config.xml">
            <!-- apply common patches here -->
        </diff>
    </diffs>


Multipatch file 'testing-multi-patch.xml':

    <diffs>
        <include file="common-multi-patch.xml"/>

        <diff file="config.xml">
            <!-- apply patches for testing environment -->
        </diff>
    </diffs>


Multipatch file 'production-multi-patch.xml':

    <diffs>
        <include file="common-multi-patch.xml"/>

        <diff file="config.xml">
            <!-- apply patches for production environment -->
        </diff>
    </diffs>


## Using the patch tool

The xml-patch library may invoked from the command line,
or from  build tools like Gradle or Ant.


### Gradle

XML patching is implemented using Gradle's built-in support for custom filters.
Any task implementing CopySpec can be augmented to perform XML patching.

File 'build.gradle':

    buildscript {
        repositories {
            jcenter()
        }

        dependencies {
            classpath "com.github.dnault:xml-patch:0.1.0"
        }
    }

    import com.github.dnault.xmlpatch.filter.XmlPatch
    import com.github.dnault.xmlpatch.filter.multi.XmlMultiPatch
    import com.github.dnault.xmlpatch.filter.multi.XmlPatchSpec

    apply plugin: 'java'

    ext {
        patchDir = "config/xml-patches"
    }

    // Copy a directory, selectively applying a single-target patch.
    //
    // Use this single-target technique if you're only patching a few documents
    // and don't require multi-patch composition.
    //
    task singleTargetPatchExample(type: Copy) {
        inputs.dir patchDir

        from "src/main/example-documents"
        into "${buildDir}/patched"

        // Use one 'filesMatching' statement for each file you want to patch.

        filesMatching("**/patch-me.xml") {
            // Single-target patches use the 'XmlPatch' filter.
            filter(XmlPatch, patch: "${patchDir}/single/first-patch.xml")
        }

        filesMatching("**/patch-me-twice.xml") {
            // Multiple filters may be applied in sequence.
            filter(XmlPatch, patch: "${patchDir}/single/first-patch.xml")
            filter(XmlPatch, patch: "${patchDir}/single/second-patch.xml")
        }
    }


    // Augment the standard 'processResources' task to apply a multi-patch
    // to classpath resources. The same technique could be applied to the 'war' task
    // to patch webapp documents.
    //
    // Use this multi-patch technique if you're patching many documents or want
    // to take advantage of multi-patch composition.
    //
    processResources {
        inputs.dir patchDir

        // To apply a multi-patch, first create a patch specification.
        def resourcesPatch = new XmlPatchSpec("${patchDir}/multi/resources-multi-patch.xml")

        filesMatching("**/*.xml") {
            // Multi-patches use the 'XMlMultiPatch' filter.
            // The filter will only process the file if the 'path' specified here
            // matches the 'file' attribute of a <diff> in the multi-patch document.
            filter(XmlMultiPatch, spec: resourcesPatch, path: it.path)
        }

        // Optionally assert that each file targeted by the patch was seen by the filter.
        doLast {
            resourcesPatch.done()
        }
    }



### Command Line

Apply an RFC 5261 patch:

    java -jar xml-patch-<version>.jar <input file> <patch file> <output file>

A dash (-) may be used to indicate standard input / output.
