# Bundling Apps

To bundle your app in an .mpk file, just make an uncompressed .zip file of it, without including the top-level `com.micropythonos.helloworld/` folder.

For example:

```
cd com.micropythonos.helloworld/
zip -r0 /tmp/com.micropythonos.helloworld_0.0.2.mpk .
```

