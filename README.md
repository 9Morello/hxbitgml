# hxbitgml

This is a fork of [hxbit](https://github.com/HeapsIO/hxbit), which adds partial support for GML, using [YAL's custom Haxe->GML compiler](https://github.com/YAL-Haxe/sfgml/). It adds small changes to hxbit so that it compiles with sfgml.

## Installation

`haxelib git hxbitgml https://github.com/9Morello/hxbitgml/`

## Use case

The main use case for this fork is to make exchanging messages between a server and a GameMaker project easier. With Haxe Enums and serialization/deserialization sharing code between the client and the server, you can use tools like pattern matching to handle them on either side in a handy way:

```haxe
enum Message {
	NewConnection;
    Disconnect;
	MoveTo(id:String, x:Int, y:Int);
    ChatMessage(senderId:String, message:String);
    // and more
}

enum SerializeError {
	InvalidData(rawData:Bytes, rawError:Dynamic);
	EmptyData;
}

@:nullSafety(Strict) @:keep class MessageTools {
	/** Helper functions to make serializing/unserializing slightly easier**/
	public static function serialize(m:Message):Result<Bytes, SerializeError> {
		if (m == null)
			return Error(EmptyData);
		try {
			var s = new hxbit.Serializer();
			var wm = new WrappedMessage(m);
			return Ok(s.serialize(wm));
		} catch (e) {
			return Error(InvalidData(Bytes.ofString(Std.string(m)), e));
		}
	}

	public static function unserialize(bytes:Bytes):Result<Message, SerializeError> {
		if (bytes == null || bytes.length < 2)
			return Error(EmptyData);
		try {
			var s = new hxbit.Serializer();
			var wm = s.unserialize(bytes, WrappedMessage);
			return Ok(wm.value);
		} catch (e) {
			return Error(InvalidData(bytes, e));
		}
	}
}

private class WrappedMessage implements hxbit.Serializable {
	@:s public var value:Message;

	public function new(v:Message) {
		this.value = v;
	}
}
```

And then, on your client/server:

```haxe
function onBytes(bytes:Bytes) {
    final maybeMessage = MessageTools.unserialize(bytes);
    if (!maybeMessage.isOk()) {
        return;
    }

    switch (maybeMessage.unwrap()) {
        
        case MoveTo(id, x, y):
            // handle move to
        
        case ChatMessage(senderId:String, message:String):
            // handle chat message
        
        // etc
        
        default:
            trace("Unknown message type");

    }
}
```

Overall, I've grown to like this approach, as you both don't have to write custom serialization/deserialization code yourself, and you don't have to deal with (potentially dynamic typed) JSON either. Using it with a protocol like WebSocket ([hxbun](https://github.com/9Morello/hxbun) supports it well) makes it very convenient to make your game client talk to your server.

This fork doesn't try to change anything besides making it compile to GML - if you're not using GameMaker, you can just use [vanilla hxbit](https://github.com/HeapsIO/hxbit) instead.

## Limitations

* When compiling to GML, you have to use the `-D sfgml_native_bytes` and `-D sf_no_opt_auto_var` directive. The first one is to make sure you're actually using native GameMaker buffers as the underlying data type for `haxe.io.Bytes`. Regarding the second one, the local variable inlining optimization will cause wrong deserialization behavior when targeting the VM runtime. The YYC target is not affected by this. You shouldn't use `-D analyzer-optimize` either if you're targeting VM, as it will override the `sf_no_opt_auto_var` definition.
* You might need to patch your local copy of `sfgml` to rename the `Int64` class on compiled code. Otherwise, you'll get a compile time error. You have to open the `gml/std/haxe/Int64.hx` file inside sfgml's library folder, and change the 10th line to this:
`@:hintType("int64") @:native("hx_int64")`

(note the `hx_int64` naming)

* sfgml doesn't support `Type.createEmptyInstance`, and I don't think it even can without new functions being added to GameMaker's runtime. This means we had to make a compromise, and use `Type.createInstance` instead. This means that, when unserializing a class, its underlying constructor gets called, which can be very undesiderable if your code has side effects inside your class's constructor. The best way to avoid issues with this is to not have side effects there besides assigning values to fields.
* none of hxbit's built-in network features are supported/tested.
* Signed Int64 support isn't implemented, although that shouldn't be hard to add.

## Documentation

Check [sfgml's](https://github.com/YAL-Haxe/sfgml) and [hxbit's](https://github.com/HeapsIO/hxbit) docs. Besides the limitations described above, there's nothing special about this fork.