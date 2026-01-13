MicroPythonOS provides a simple way to load and save preferences, similar to Android's "SharedPreferences" framework.

Here's a simple example of how to add it to your app, taken from [QuasiNametag](https://github.com/QuasiKili/MPOS-QuasiNametag):

<pre>
```
--- quasinametag.py.orig        2025-10-29 12:24:27.494193748 +0100
+++ quasinametag.py     2025-10-29 12:07:59.357264302 +0100
@@ -1,4 +1,5 @@
 from mpos import Activity
+import mpos.config
 import mpos.ui
 import mpos.ui.anim
 import mpos.ui.focus_direction
@@ -42,6 +43,12 @@
         # Add key event handler to container to catch all key events
         container.add_event_cb(self.global_key_handler, lv.EVENT.KEY, None)
 
+        print("Loading preferences...")
+        prefs = mpos.config.SharedPreferences("com.quasikili.quasinametag")
+        self.name_text = prefs.get_string("name_text", self.name_text)
+        self.fg_color = prefs.get_int("fg_color", self.fg_color)
+        self.bg_color = prefs.get_int("bg_color", self.bg_color)
+
         # Create both screens as children of the container
         self.create_edit_screen(container)
         self.create_display_screen(container)
@@ -263,6 +270,13 @@
         if focusgroup:
             mpos.ui.focus_direction.emulate_focus_obj(focusgroup, self.display_screen)
 
+        print("Saving preferences...")
+        editor = mpos.config.SharedPreferences("com.quasikili.quasinametag").edit()
+        editor.put_string("name_text", self.name_text)
+        editor.put_int("fg_color", self.fg_color)
+        editor.put_int("bg_color", self.bg_color)
+        editor.commit()
+
     def update_display_screen(self):
         # Set background color
         self.display_screen.set_style_bg_color(lv.color_hex(self.bg_color), 0)
```
</pre>

Here's a more complete example:

<pre>
```
# Example usage with access_points as a dictionary
def main():
    # Initialize SharedPreferences
    prefs = SharedPreferences("com.example.test_shared_prefs")

    # Save some simple settings and a dictionary-based access_points
    editor = prefs.edit()
    editor.put_string("someconfig", "somevalue")
    editor.put_int("othervalue", 54321)
    editor.put_dict("access_points", {
        "example_ssid1": {"password": "examplepass1", "detail": "yes please", "numericalconf": 1234},
        "example_ssid2": {"password": "examplepass2", "detail": "no please", "numericalconf": 9875}
    })
    editor.apply()

    # Read back the settings
    print("Simple settings:")
    print("someconfig:", prefs.get_string("someconfig", "default_value"))
    print("othervalue:", prefs.get_int("othervalue", 0))

    print("\nAccess points (dictionary-based):")
    ssids = prefs.get_dict_keys("access_points")
    for ssid in ssids:
        print(f"Access Point SSID: {ssid}")
        print(f"  Password: {prefs.get_dict_item_field('access_points', ssid, 'password', 'N/A')}")
        print(f"  Detail: {prefs.get_dict_item_field('access_points', ssid, 'detail', 'N/A')}")
        print(f"  Numerical Conf: {prefs.get_dict_item_field('access_points', ssid, 'numericalconf', 0)}")
        print(f"  Full config: {prefs.get_dict_item('access_points', ssid)}")

    # Add a new access point
    editor = prefs.edit()
    editor.put_dict_item("access_points", "example_ssid3", {
        "password": "examplepass3",
        "detail": "maybe",
        "numericalconf": 5555
    })
    editor.commit()

    # Update an existing access point
    editor = prefs.edit()
    editor.put_dict_item("access_points", "example_ssid1", {
        "password": "newpass1",
        "detail": "updated please",
        "numericalconf": 4321
    })
    editor.commit()

    # Remove an access point
    editor = prefs.edit()
    editor.remove_dict_item("access_points", "example_ssid2")
    editor.commit()

    # Read updated access points
    print("\nUpdated access points (dictionary-based):")
    ssids = prefs.get_dict_keys("access_points")
    for ssid in ssids:
        print(f"Access Point SSID: {ssid}: {prefs.get_dict_item('access_points', ssid)}")

    # Demonstrate compatibility with list-based configs
    editor = prefs.edit()
    editor.put_list("somelist", [
        {"a": "ok", "numericalconf": 1111},
        {"a": "not ok", "numericalconf": 2222}
    ])
    editor.apply()

    print("\List-based config:")
    somelist = prefs.get_list("somelist")
    for i, ap in enumerate(somelist):
        print(f"List item {i}:")
        print(f"  a: {prefs.get_list_item('somelist', i, 'a', 'N/A')}")
        print(f"  Full dict: {prefs.get_list_item_dict('somelist', i)}")

if __name__ == '__main__':
    main()
```
</pre>
