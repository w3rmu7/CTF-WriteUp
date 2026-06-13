1. Overview

The challenge used a two-stage Java server execution flow based on an AOT cache file. In the first stage, the server accepted an uploaded cache file and saved it as /tmp/cache.aot. In the second stage, the real leftovers2.jar application was executed using that uploaded file as its AOT cache.

The vulnerability was that the second-stage application trusted runtime state coming from an attacker-controlled AOT cache. By patching the archived string object "images" inside the cache to "//////", the default image directory of the application could be changed from images to the filesystem root /.

After that, requesting the image for a product named flag caused the server to read /flag.

The final flag was:

GPNCTF{1_hoPE_ThE_cAchE_iS_neVER_pr0vidED_bY_Li8R4R1eS}
2. Execution Structure

The exec.sh script used a two-stage structure.

In stage 1, the outer server was executed using an AOT cache:

OuterServer serve /tmp/cache.aot cache.aot

This stage allowed the attacker to interact with OuterServer.

In stage 2, the file saved at:

/tmp/cache.aot

was reused as the AOT cache for the actual leftovers2.jar server.

Therefore, the uploaded cache in stage 1 directly influenced the runtime behavior of the application in stage 2.

3. Stage 1 Analysis

The first-stage AOT cache, outer-cache.aot, contained the class:

de.kitctf.gpn24.leftovers2.OuterServer

By using JVMTI to dump class bytecode loaded from the AOT cache, the routes implemented by OuterServer could be recovered.

The important routes were:

GET /cache
POST /init

The GET /cache route returned the original cache.aot file.

The POST /init route accepted an uploaded cache file, saved it as:

/tmp/cache.aot

and then proceeded to the second stage.

This meant the attacker could first obtain the original cache, patch it locally, and upload the modified version through /init.

4. Stage 2 Application Analysis

The second-stage application behaved similarly to the previous leftovers challenge. It exposed routes such as:

PUT /products/{name}
GET /images/{name}
POST /set-image-dir

At first, /set-image-dir seemed like the obvious target because changing the image directory would allow path redirection. However, in this challenge, that functionality was intentionally disabled.

The password check inside the AOT cache effectively returned:

return Boolean.FALSE;

The server also returned the following message:

Password login is currently disabled

This showed that the intended solution was not to recover or brute-force a password. Instead, the important attack surface was the AOT cache itself.

5. Root Cause

When the real application started, the image store was initialized with the default directory:

new ImageStore(Path.of("images"))

The image retrieval logic was roughly:

imageFolder.resolve(sanitizeName(product.name()))

Therefore, if the image directory was the default value images, requesting the image of a product named flag would make the server try to open:

images/flag

However, if the image directory could be changed to /, the same request would instead resolve to:

/flag

Since the remote server had the real flag at /flag, changing the default image directory to / would make GET /images/flag return the flag.

6. AOT Cache Patching

The goal was to change the default image directory from:

images

to:

/

However, directly replacing images with / would change the string length, which could corrupt the cache structure. The replacement had to preserve the original six-byte length.

The chosen replacement was:

images -> //////

This is still six bytes long.

In Java, the expression:

Path.of("//////")

is normalized to the filesystem root:

/

Therefore, replacing "images" with "//////" changed the application’s default image directory to the root directory while preserving the binary layout of the AOT cache.

7. Finding the Correct String Locations

The first attempt was to patch the string images found in the metadata area of cache.aot. However, changing only this occurrence did not affect the application’s runtime behavior.

Further analysis showed that the actual archived String object used at runtime was stored separately in a later string table.

Two relevant occurrences of the six-byte string images were found:

metadata occurrence:        21390686
archived string occurrence: 52751232

Both occurrences had to be patched.

The final patch changed both of them from:

images

to:

//////

The important part was that the archived runtime string, not only the metadata string, had to be modified.

8. Local Verification

For local testing, creating /flag was not possible due to permission restrictions. Therefore, the same technique was tested using /tmp/flag.

Instead of patching:

images -> //////

the local test used:

images -> /tmp//

This also preserved the original six-byte length.

Then a test flag file was created at:

/tmp/flag

After uploading the patched cache through the stage 1 /init route, the stage 2 application started with /tmp// as its image directory.

Then the following flow was tested:

PUT /products/flag
GET /images/flag

Because the image directory had been changed to /tmp//, requesting /images/flag caused the application to read:

/tmp/flag

This confirmed that patching the archived string inside the AOT cache successfully controlled the image directory used by the application.

9. Remote Exploitation

On the remote server, the actual flag existed at:

/flag

Therefore, the final cache patch was:

images -> //////

After patching both the metadata occurrence and the archived runtime string occurrence, the modified cache was uploaded to the stage 1 server:

POST /init

The server saved the uploaded cache as:

/tmp/cache.aot

Then stage 2 started using the attacker-controlled cache.

After stage 2 was running, a product named flag was created:

PUT /products/flag

Then its image was requested:

GET /images/flag

Since the default image directory had been changed to /, the application resolved the requested image path to:

/flag

The response contained the flag:

GPNCTF{1_hoPE_ThE_cAchE_iS_neVER_pr0vidED_bY_Li8R4R1eS}
10. Exploit Summary

The full exploit flow was:

Analyze exec.sh and identify the two-stage execution structure.
Inspect outer-cache.aot.
Dump the class bytecode loaded from the AOT cache using JVMTI.
Recover the important OuterServer routes:
GET /cache
POST /init
Download the original cache.aot.
Analyze the second-stage application routes.
Confirm that /set-image-dir was disabled.
Identify that the image store was initialized with Path.of("images").
Find both relevant occurrences of the string images in the AOT cache:
metadata occurrence at 21390686
archived runtime string occurrence at 52751232
Patch both occurrences:
images -> //////
Upload the patched cache through /init.
Let the application continue to stage 2.
Create a product named flag.
Request /images/flag.
Read /flag through the modified image directory.
11. Conclusion

The challenge was not solved by exploiting the normal /set-image-dir endpoint, because that functionality was disabled. Instead, the actual vulnerability was that the application reused an attacker-provided AOT cache as trusted runtime state.

By modifying the archived String("images") object inside the AOT cache to String("//////"), the default image directory was changed from images to /.

This caused the existing image retrieval logic to read /flag when the attacker requested the image for a product named flag.

The core issue was that the AOT cache was treated as a trusted part of the application, even though it could be supplied by the attacker during stage 1.

Final flag:

GPNCTF{1_hoPE_ThE_cAchE_iS_neVER_pr0vidED_bY_Li8R4R1eS}