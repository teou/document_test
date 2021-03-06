Run the following eval action test, a panic "should never get here" will be raised. 
```
func TestEvalActionWithInterfaceRcv(t *T)  {
	getSet := NewEvalScript(1, `
		local prev = redis.call("GET", KEYS[1])
		redis.call("SET", KEYS[1], ARGV[1])
		return 123
		-- `+randStr() /* so there's an eval everytime */ +`
	`)

	c := dial()
	key := randStr()
	val := randStr()
	{
		var res interface{}
		err := c.Do(getSet.Cmd(&res, key, val))
		require.Nil(t, err)
		assert.Equal(t, 123, res)
	}
}
```
The panic stack info.
```
=== RUN   TestEvalActionWithInterfaceRcv
--- FAIL: TestEvalActionWithInterfaceRcv (0.00s)
panic: should never get here [recovered]
	panic: should never get here

goroutine 7 [running]:
testing.tRunner.func1(0xc0003a6100)
	/usr/local/go/src/testing/testing.go:874 +0x3a3
panic(0x7551c0, 0x86b7c0)
	/usr/local/go/src/runtime/panic.go:679 +0x1b2
github.com/mediocregopher/radix/v3/resp/resp2.saneDefault(...)
	/home/code/radix/resp/resp2/resp.go:789
github.com/mediocregopher/radix/v3/resp/resp2.Any.UnmarshalRESP(0x744de0, 0xc0003c6010, 0x0, 0xc0003ba060, 0xc0000d1d40, 0x40b828)
	/home/code/radix/resp/resp2/resp.go:828 +0xc1f
github.com/mediocregopher/radix/v3.(*connWrap).Decode(0xc0003c0020, 0x8738e0, 0xc0003c0060, 0xc0003c0060, 0x0)
	/home/code/radix/conn.go:93 +0x42
github.com/mediocregopher/radix/v3.(*evalAction).Run.func1(0x0, 0xc0003ca000, 0xc0003b80a0)
	/home/code/radix/action.go:489 +0xfa
github.com/mediocregopher/radix/v3.(*evalAction).Run(0xc0003ca000, 0x87b320, 0xc0003c0020, 0xc0003b8001, 0xc0003ca000)
	/home/code/radix/action.go:492 +0x68
github.com/mediocregopher/radix/v3.(*connWrap).Do(0xc0003c0020, 0x876640, 0xc0003ca000, 0xe, 0x0)
	/home/code/radix/conn.go:82 +0x47
github.com/mediocregopher/radix/v3.TestEvalActionWithInterfaceRcv(0xc0003a6100)
	/home/code/radix/action_test.go:221 +0x322
testing.tRunner(0xc0003a6100, 0x7f8f68)
	/usr/local/go/src/testing/testing.go:909 +0xc9
created by testing.(*T).Run
	/usr/local/go/src/testing/testing.go:960 +0x350

Process finished with exit code 1
```

Seems if we set the response receiver for EvalAction.Cmd to interface{}, when the EvalAction runs `run(false)` for the first time, a panic "should never get here" will be raised, for returned "NOSCRIPT" error message is not expected in function saneDefault. 
```
 func (ec *evalAction) Run(conn Conn) error {
	run := func(eval bool) error {
		ec.eval = eval
		if err := conn.Encode(ec); err != nil {
			return err
		}
		return conn.Decode(resp2.Any{I: ec.rcv})
	}

	err := run(false)
	if err != nil && strings.HasPrefix(err.Error(), "NOSCRIPT") {
		err = run(true)
	}
	return err
} 
```

```
func (a Any) UnmarshalRESP(br *bufio.Reader) error {
	// if I is itself an Unmarshaler just hit that directly
	if u, ok := a.I.(resp.Unmarshaler); ok {
		return u.UnmarshalRESP(br)
	}

	b, err := br.Peek(1)
	if err != nil {
		return err
	}
	prefix := b[0]

	if prefix == ErrorPrefix[0] {
		return Error{E: errors.New(string(b))}
	}
	
	// This is a super special case that _must_ be handled before we actually
	// read from the reader. If an *interface{} is given we instead unmarshal
	// into a default (created based on the type of th message), then set the
	// *interface{} to that
	if ai, ok := a.I.(*interface{}); ok {
		innerA := Any{I: saneDefault(prefix)}
		if err := innerA.UnmarshalRESP(br); err != nil {
			return err
		}
		*ai = reflect.ValueOf(innerA.I).Elem().Interface()
		return nil
	}

	br.Discard(1)
	b, err = bytesutil.BufferedBytesDelim(br)
	if err != nil {
		return err
	}

	switch prefix {
	case ErrorPrefix[0]:
		return Error{E: errors.New(string(b))}
	case ArrayPrefix[0]:
		l, err := bytesutil.ParseInt(b)
		if err != nil {
			return err
		} else if l == -1 {
			return a.unmarshalNil()
		}
		return a.unmarshalArray(br, l)
	case BulkStringPrefix[0]:
		l, err := bytesutil.ParseInt(b) // fuck DRY
		if err != nil {
			return err
		} else if l == -1 {
			return a.unmarshalNil()
		}

		// This is a bit of a clusterfuck. Basically:
		// - If unmarshal returns a non-Discarded error, return that asap.
		// - If discarding the last 2 bytes (in order to discard the full
		//   message) fails, return that asap
		// - Otherwise return the original error, if there was any
		if err = a.unmarshalSingle(br, int(l)); err != nil {
			if !errors.As(err, new(resp.ErrDiscarded)) {
				return err
			}
		}
		if _, discardErr := br.Discard(2); discardErr != nil {
			return discardErr
		}
		return err
	case SimpleStringPrefix[0], IntPrefix[0]:
		reader := byteReaderPool.Get().(*bytes.Reader)
		reader.Reset(b)
		err := a.unmarshalSingle(reader, reader.Len())
		byteReaderPool.Put(reader)
		return err
	default:
		return errors.Errorf("unknown type prefix %q", b[0])
	}
}
```
When the eval script sha is not in redis, the result retured by redis is `-NOSCRIPT` whose prefix is '-',  but '-' in the `saneDefault` is not handled and and throws a panic "should never get here", as the following code.
```
func saneDefault(prefix byte) interface{} {
	// we don't handle ErrorPrefix because that always returns an error and
	// doesn't touch I
	switch prefix {
	case ArrayPrefix[0]:
		ii := make([]interface{}, 8)
		return &ii
	case BulkStringPrefix[0]:
		bb := make([]byte, 16)
		return &bb
	case SimpleStringPrefix[0]:
		return new(string)
	case IntPrefix[0]:
		return new(int64)
	}
	panic("should never get here")
}
```

the following code won't throw a panic any more and can return correct error msg when response receiver is interface{} in evalAction, and will return the value when redis is not returning errors.
```
	// This is a super special case that _must_ be handled before we actually
	// read from the reader. If an *interface{} is given we instead unmarshal
	// into a default (created based on the type of th message), then set the
	// *interface{} to that
	if ai, ok := a.I.(*interface{}); ok && prefix != ErrorPrefix[0] {
		innerA := Any{I: saneDefault(prefix)}
		if err := innerA.UnmarshalRESP(br); err != nil {
			return err
		}
		*ai = reflect.ValueOf(innerA.I).Elem().Interface()
		return nil
	}
```
