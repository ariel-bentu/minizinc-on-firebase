# 🧩 Running MiniZinc in Firebase Cloud Functions

This project demonstrates how to run a native MiniZinc model inside a Firebase Cloud Function.

## 🚀 Motivation

MiniZinc is a powerful constraint modeling language, but deploying it in serverless environments like Firebase is challenging due to native binaries, dynamic libraries, and symbolic links. This project shows how to overcome these challenges and enables constraint-solving in scalable, cloud-native applications—for example, automated group assignments or scheduling problems—without needing dedicated backend infrastructure.

## 🛠️ Preparing the MiniZinc Package

Before deploying to Firebase, we need to strip the MiniZinc distribution down to its essentials and replace symlinks with actual files (Firebase does not support symlinks in deployments):

- Download the Linux Binary Archive. found [here](https://www.minizinc.org/downloads/)
- Expand the tar and rename the folder to minizinc
- Execute the following:

```bash
#!/bin/bash

cd minizinc/lib || exit 1

# Remove large or unused shared libraries
rm ./libicudata.so.73
rm ./libQt6Gui.so.6
rm ./libQt6Widgets.so.6
rm ./libQt6Core.so.6
rm ./libprotoc.so.29.3.0
rm ./libcrypto.so.3
rm ./libicui18n.so.73
rm ./libhighs.so

# Replace symbolic links with the actual library binaries
for symlink in *.so; do
  if [ -L "$symlink" ]; then
    target=$(readlink "$symlink")
    echo "Copying real contents of $target to $symlink (replacing symlink)"
    cp -f --remove-destination "$target" "$symlink"
  fi
done

# Remove unused binaries
cd ../bin
rm MiniZincIDE
rm findMUS
rm fzn-gecode
rm minizinc-globalizer

cd ..
rm -rf plugins
```

## Packaging
- Place the resulting minizinc/ folder inside your functions/ directory in Firebase.
- Place your *.mzn file (e.g. `your-model.mzn`) next to you index.js

## A Firebase function example

This example functions assume the model is deployed with your function as `./your-model.mzn`. It expects as an input the input data as string. 

The Payload to the function is:
```typescript
interface SolvePayload {
    data: string;
    timeout?: number; // in milliseconds
}
```



```javascript
const { spawn } = require("child_process");
const fs = require("fs");
const path = require("path");
const os = require("os");
const functions = require("firebase-functions");

exports.solve = functions.region("europe-west1")
    .https.onCall((data, context) => {
        const user = context.auth.token.email;
        if (!user || user.length === 0) {
            throw new Error("Unauthorized");
        }

        const timeout = data.timeout;
        const modelFile = path.join(__dirname, "your-model.mzn");
        const stamp = new Date().toISOString().replace(/[:T]/g, "-").replace(/\..+/, "");
        const dataFile = path.join(os.tmpdir(), `${stamp}.dzn`);
        fs.writeFileSync(dataFile, data.data);

        functions.logger.info("Minizinc", modelFile, dataFile, data.data);

        return new Promise((resolve) => {
            const proc = spawn("minizinc/bin/minizinc", [
                modelFile,
                dataFile,
                "--time-limit", `${timeout}`,
                "--solver", "cp-sat",
            ], {
                cwd: __dirname,
                env: {
                    ...process.env,
                    LD_LIBRARY_PATH: path.join(__dirname, "minizinc", "lib"),
                },
            });

            let output = "";
            let error = "";

            proc.stdout.on("data", (data) => output += data.toString());
            proc.stderr.on("data", (data) => error += data.toString());

            proc.on("close", (code) => {
                if (code === 0) {
                    functions.logger.info("Minizinc done", output);
                    resolve({ result: mznStdoutToJson(output) });
                } else {
                    resolve({ result: getErrorJson(error) });
                }
            });
        });
    });

    function mznStdoutToJson(output) {
        // todo
        return {success: true}
    }

    function getErrorJson(output) {
        // todo
        return {success: false}
    }
```
