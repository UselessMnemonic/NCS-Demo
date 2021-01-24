# Project Documentation

## Getting Started 

I recommend letting the [Homebrew](https://brew.sh/) package manager take care of software.

I installed both IntelliJ and Eclipse, however IntelliJ has proven easier to use. Eclipse does not recognize the environment variables set up later in this guide when invoking Ant targets.

My setup is off a fresh new MacBook Pro running macOS Big Sur.

macOS now uses zsh for its interactive shell. I created a `.zshenv` file which will contain environment variables, similar to `.bashrc`, for all invocations of the shell. If you already have a stable environment setup, adapt the following guide for your situation.

If you cloned this repo and want to play with it, make sure you:

- symlink the cisco-ios-cli-3.8 package into the `packages` folder. You can do this with `ln -s $NCS_DIR/packages/neds/cisco-ios-cli-3.8/`

- run `ncs-setup --dest .` in the root, otherwise the virtual network might not appear inside ncs

- `make` the sample packages in `package`, otherwise ncs will fail to load them/start

### Prerequisites:

- Add the AdoptOpenJDK repository with `brew tap AdoptOpenJDK/openjdk`
- Install Java 11 with `brew install adoptopenjdk11`
- Install Ant with `brew install ant` 
- Install Python 3.9 with `brew install python@3.9`
- Install IntelliJ with `brew install intellij-idea-ce`
- I also recommend installing VSCode with `brew install visual-studio-code`
- If you also want to install Eclipse, you can with `brew install eclipse-java`
- If you plan to use the lux testing framework, `brew tap hawk/homebrew-hawk` and `brew install lux`

### NSO Setup:

- Download and install an NSO package (I used `nso-5.4.2.darwin.x86_64`, which corresponds to my local installation folder `~/ncs-5.4.2`).
  
  Remember to use `sh nso-VERSION.OS.ARCH.installer.bin ~/ncs-VERSION --local-install` where `VERSION` is the version number and `OS`/`ARCH` correspond to your machine.

- Add the following to `.zshenv`:

  `source ~/ncs-5.4.2/ncsrc`

  You might need `python` and `pip` to alias to `python3` and `pip3` respectively, so add these lines as well: 
  
  `alias python=/usr/local/bin/python3`
  
  `alias pip=/usr/local/bin/pip3`

### First project:

- In the shell, navigate to where you would like to create your project (for example, I place my demo code in `~/Documents/NCS-Projects/DemoService`)

- Generate a simulated network with `ncs-netsim create-network cisco-ios-cli-3.8 3 ios`, and the project files with `ncs-setup --dest .`

- Navigate to `netsim/ios` and observe that there are 3 new virtual devices: `ios0`, `ios1`, and `ios2`. From here, you can start the simulator with `ncs-netsim start`, then navigate back to the project root.

- Enter the `packages` folder, run `ncs-make-package --help`, and observe possible project types. We will be using the `java-and-template` skeleton, but you could also use the `python-and-template` skeleton. Use `ncs-make-package --service-skeleton java-and-template example1` to create a package `example1`, then navigate into it

- Note the `src` and `templates` folders. Inside `src` are also `java` and `yang` folders. All these contain the java classes, yang configurations, and xml templates needed to design a service. 

## Development Example

### Using IntelliJ

Starting development is as easy as creating a project. Open up IntelliJ and create a project, with the `packages/example1/src/java` folder as your root.

You might see a package called `namespaces` is missing. Under the `src` folder is a Makefile. You can run `make` here to generate that file based off the YANG configuration in the `yang` foler. Always `make` when tha YANG configuration is modified.

You might also notice that none of the imported libraries are available. To import them:

- Navigate to `File > Project Structure > Global Libraries` and create a new Java library. 

- In the file dialogue, navigate to `~/ncs-5.4.2/java/jar` and select all. Accept the next dialogue.

- Navigate to the `Modules > Dependencies` tab and add the new library. The IDE will now recognize the imports

The `build.xml` file is an Ant script. You can invoke the build targets from within IntelliJ to rebuild the project as you modify it.

### Using VSCode

I use VSCode to modify YANG files, using the [Yangster](https://marketplace.visualstudio.com/items?itemName=typefox.yang-vscode) plugin. You can create a file called `yang.settings` either in the `yang` folder or in a folder on your home like `~/.yang/yang.settings`. This will allow the plugin to recognize the imported types. The file is in JSON, and should contain an absolute path like the following:

```json
{
  "yangPath": "/Users/chmadrig/ncs-5.4.2/src/ncs/yang"
}
```

### Porting a Python example to Java

We can implement projects in Java, or Python, or even just with templates. Since my skillset is largely within Java, I decided to start the project by porting an example Python-based service into Java.

The basic configuration is the same. The only challenge is porting the servicepoint. Let's look at the sample, starting with parts of the YANG and XML:

```yang
list example1 {
  description "This is an RFS skeleton service";

  // modified from the skeleton
  key community;
  leaf community {
    tailf:info "community string";
    type string;
  }

  ...

  // modified from the skeleton
  leaf access_right {
    tailf:info "read-only or read-write";
    type enumeration {
      enum read-only;
      enum read-write;
    }
  }
}
```

```xml
<config>
  <snmp-server xmlns="urn:ios">
    <community>
      <name>{$COMMUNITY}</name>
      <RW when="{starts-with($ACCESS, 'RW')}" />
      <RO when="{starts-with($ACCESS, 'RO')}" />
    </community>
  </snmp-server>
</config>
```

The YANG file describes two leaves: one for a property called `community` and another for a property called `access_right`. Similarly, the XML contains references to some variables called `COMMUNITY` and `ACCESS`. Take a look at the Python script:

```python
vars = ncs.template.Variables()
# This COMMUNITY and ACCESS is passed to XML
vars.add('COMMUNITY', service.community + str(date.today().year))
vars.add('ACCESS', 'ro' if service.access_right == 'read-only' else 'rw')
template = ncs.template.Template(service)
template.apply('custom_snmp_community-template', vars)
```

The YANG defines the service, and so the user will be able to see those leaves as parameters. The Python script takes the values provided by the user and applies some transformation--that is the power of an intermediate processor. In this case, we append the current year to the value for `community`.

Finally, the new values are applied to the template via the `vars` container. We can achieve the same thing in Java:

```java
Template myTemplate = new Template(context, "example1-template");
TemplateVariables myVars = new TemplateVariables();

// from YANG configuration
String community = service.leaf("community").valueAsString();
String access_right = service.leaf("access_right").valueAsString();
int year = Calendar.getInstance().get(Calendar.YEAR);

try {
    myVars.putQuoted("COMMUNITY", community + year);
    myVars.putQuoted("ACCESS", access_right.equals("read-only") ? "RO" : "RW");
    myTemplate.apply(service, myVars);

} catch (Exception e) {
    throw new DpCallbackException(e.getMessage(), e);
}
```

After you are satisfied with the YANG, `make all` in the Makefile under the `src` folder. This will regenerate any auto-generated files as well as invoke Ant to compile the Java source.

Start NCS by running `ncs` in the root folder where we initialized the project. Then, we can look at the service by entering the CLI with `ncs_cli -u admin`:

```
% ncs
% ncs_cli -u admin
admin@ncs> show packages package oper-status 
packages package cisco-ios-cli-3.8
 oper-status up
packages package example1
 oper-status up
[ok]
```

To test the serivice, you can enter configuration mode and use the service to modify a device:

```
admin@ncs> configure 
Entering configuration mode private
[ok]
admin@ncs% set example1 randomCommunity access_right read-only device ios0
[ok]
admin@ncs% commit dry-run outformat native 
native {
    device {
        name ios0
        data snmp-server community randomCommunity2021 RO
    }
}
[ok]
```

You can see the transformation was applied. The service performs the same function as:

```
admin@ncs% set devices device ios0 config snmp-server community randomCommunity2021 RO
[ok]
admin@ncs% commit dry-run outformat native                                            
native {
    device {
        name ios0
        data snmp-server community randomCommunity2021 RO
    }
}
[ok]
``` 

## Second Example 

I started a simple firewall congiuration service from scratch, using the `java-and-template` skeleton and following the same steps as above. The service maintains a list of blocked hosts and applies the rules to each selected device.