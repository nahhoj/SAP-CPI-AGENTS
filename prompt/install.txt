Perform the following tasks in order:

1. Detect the operating system currently running (Windows, macOS, or Linux).

2. Check if VSCode is installed. If it is not, install it from:
   https://code.visualstudio.com/Download

3. Check and install the following extension:
   - VSCode extension "SAP CPI Groovy Script":
     https://marketplace.visualstudio.com/items?itemName=JohanCalderon.sap-cpi-groovy-script

4. Download:
   - Java JDK 17.0.19: https://learn.microsoft.com/en-us/java/openjdk/older-releases
   - Groovy 4.0.29 (binary release): https://archive.apache.org/dist/groovy

5. Extract each downloaded package (Java and Groovy) into the following folder, creating it if it doesn't exist:
   - ~/programs

6. In VSCode's settings (settings.json), add the following keys. For each one, use the root path of the extracted folder — the one that directly contains the "bin" subfolder — NOT the path to "bin" itself:
   - groovy.java.home: <root path of the extracted Java folder, i.e. the folder containing "bin">
   - groovy.groovy.home: <root path of the extracted Groovy folder, i.e. the folder containing "bin">

7. download agent and skill:
   - agent: https://github.com/nahhoj/SAP-CPI-AGENTS/blob/master/agent/sap-cpi-groovy.md
   - skill: https://github.com/nahhoj/SAP-CPI-AGENTS/blob/master/skill/sap-cpi-groovy-best-practice/SKILL.md

8. add the agent and skill to ~/.config/opencode:
   - agent: ~/.config/opencode/agent/sap-cpi-groovy.md
   - skill: ~/.config/opencode/skill/sap-cpi-groovy-best-practice/SKILL.md

9. Delete all files downloaded (clean)
