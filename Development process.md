Development process
Even if your WeWeb components are standard Vue components, they need to be loaded inside a specific context to be tested. The easiest way to test is to use the WeWeb Editor in development mode. Here is a description of the different steps to test and develop your component.

Initiate your directory
Compatibility note

weweb-cli requires Node.js version >=18.0.0. weweb.io use Vue 3

We provide a npm template to quickly start development.

For npm

shell
npm init @weweb/component
For yarn

shell
yarn create @weweb/component
Then you have to follow the different prompt questions.

If you already know what you want, you can pass the component name and the type of component you want via additional command line options.

For example, if you want to create an element named my-element or a section named hero-section:

shell
# npm 6.x (use npm -v to know your version)
npm init @weweb/component my-element --type element
npm init @weweb/component hero-section --type section

# npm 7+, extra double-dash is needed (use npm -v to know your version)
npm init @weweb/component my-element -- --type element
npm init @weweb/component hero-section -- --type section

# yarn
yarn create @weweb/component my-element --type element
yarn create @weweb/component hero-section --type section
The directory will contain:

a src directory with a file for your component (wwSection.vue or wwElement.vue). Available props and events are listed here
a package.json with two scripts for dev and build, and weweb/cli as dependencies
a basic README.md
a ww-config.js describing all the properties of your component. Learn more about it here
WARNING

You can add other dependencies if you need to. Be aware that all package.json will be merged in the final web-app and, if two components need the same dependencies, only the more recent one will be installed. This can lead to conflicts.

DANGER

You do not need vue as a dependency, as it will be already provided by the Editor or the application.

Install dependencies and start dev server
Go to the directory you just created, install dependencies and start a dev server.

For npm:

shell
# Install dependencies
npm install

# Start with default 8080 port
npm run serve

# Or give a port (usefull if you are developing several components)
npm run serve -- port=4040
For yarn:

shell
# Install dependencies
yarn

# Start with default 8080 port
yarn serve

# Or give a port (usefull if you are developing several components)
yarn serve -- port=4040
We use https to serve components. Your browser needs to authorize this.

Go to https://localhost:8080 (or the port you choose).

Click on Advanced Settings then on Continue to localhost

Chrome settings

If you do not see Advanced settings, type chrome://flags as a URL in your browser. Search for Allow invalid certificates for resources loaded from localhost and click Enable