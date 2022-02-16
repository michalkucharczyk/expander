# expander

Expands a proc-macro into a file, and uses a `include!` directive in place.


## Advantages

* Only expands a particular proc-macro, not all of them. I.e. `tracing` is notorious for expanding into a significant amount of boilerplate with i.e. `cargo expand`
* Get good errors when _your_ generated code is not perfect yet


## Usage

In your `proc-macro`, use it like:

```rust

#[proc_macro_attribute]
pub fn baz(_attr: proc_macro::TokenStream, input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    // wrap as per usual for `proc-macro2::TokenStream`, here dropping `attr` for simplicity
    baz2(input.into()).into()
}


 // or any other macro type
fn baz2(input: proc_macro2::TokenStream) -> proc_macro2::TokenStream {
    let modified = quote::quote!{
        #[derive(Debug, Clone, Copy)]
        #input
    };

    let expanded = Expander::new("baz")
        .add_comment("This is generated code!".to_owned())
        .fmt(Edition::_2021)
        // common way of gating this, by making it part of the default feature set
        .dry(cfg!(feature="no-file-expansion"))
        .write_to_out_dir(modified.clone()).unwrap_or_else(|e| {
            eprintln!("Failed to write to file: {:?}", e);
            modified
        });
    expanded
}
```

will expand into

```rust
include!("/absolute/path/to/your/project/target/debug/build/expander-49db7ae3a501e9f4/out/baz-874698265c6c4afd1044a1ced12437c901a26034120b464626128281016424db.rs");
```

where the file content will be

```rust
#[derive(Debug, Clone, Copy)]
struct X {
    y: [u8:32],
}
```


# Examplary output

An error in your proc-macro, i.e. an excess `;`, is shown as

---

<pre><font color="#26A269"><b>   Compiling</b></font> expander v0.0.4-alpha.0 (/somewhere/expander)
<font color="#F66151"><b>error</b></font><b>: macro expansion ignores token `;` and any following</b>
 <font color="#2A7BDE"><b>--&gt; </b></font>tests/multiple.rs:1:1
  <font color="#2A7BDE"><b>|</b></font>
<font color="#2A7BDE"><b>1</b></font> <font color="#2A7BDE"><b>| </b></font>#[baz::baz]
  <font color="#2A7BDE"><b>| </b></font><font color="#F66151"><b>^^^^^^^^^^^</b></font> <font color="#F66151"><b>caused by the macro expansion here</b></font>
  <font color="#2A7BDE"><b>|</b></font>
  <font color="#2A7BDE"><b>= </b></font><b>note</b>: the usage of `baz::baz!` is likely invalid in item context

<font color="#F66151"><b>error</b></font><b>: macro expansion ignores token `;` and any following</b>
 <font color="#2A7BDE"><b>--&gt; </b></font>tests/multiple.rs:4:1
  <font color="#2A7BDE"><b>|</b></font>
<font color="#2A7BDE"><b>4</b></font> <font color="#2A7BDE"><b>| </b></font>#[baz::baz]
  <font color="#2A7BDE"><b>| </b></font><font color="#F66151"><b>^^^^^^^^^^^</b></font> <font color="#F66151"><b>caused by the macro expansion here</b></font>
  <font color="#2A7BDE"><b>|</b></font>
  <font color="#2A7BDE"><b>= </b></font><b>note</b>: the usage of `baz::baz!` is likely invalid in item context

<font color="#C01C28"><b>error</b></font><b>:</b> could not compile `expander` due to 2 previous errors
<font color="#FFBF00"><b>warning</b></font><b>:</b> build failed, waiting for other jobs to finish...
<font color="#C01C28"><b>error</b></font><b>:</b> build failed
</pre>

---

becomes

---

<pre>
<font color="#26A269"><b>   Compiling</b></font> expander v0.0.4-alpha.0 (/somewhere/expander)
dest: /somewhere/expander/target/debug/build/expander-8cb9d7a52d4e83d1/out/baz-874698265c6c4afd1044a1ced12437c901a26034120b464626128281016424db.rs
<font color="#F66151"><b>error</b></font><b>: expected item, found `;`</b>
 <font color="#2A7BDE"><b>--&gt; </b></font>/somewhere/expander/target/debug/build/expander-8cb9d7a52d4e83d1/out/baz-874698265c6c4afd1044a1ced12437c901a26034120b464626128281016424db.rs:2:42
  <font color="#2A7BDE"><b>|</b></font>
<font color="#2A7BDE"><b>2</b></font> <font color="#2A7BDE"><b>| </b></font>#[derive(Debug, Clone, Copy)] struct A ; ;
  <font color="#2A7BDE"><b>| </b></font>                                         <font color="#F66151"><b>^</b></font>

dest: /somewhere/expander/target/debug/build/expander-8cb9d7a52d4e83d1/out/baz-73b3d5b9bc4641a894d85b878020da6e331ecfd8949c9157c50a92845412a534.rs
<font color="#F66151"><b>error</b></font><b>: expected item, found `;`</b>
 <font color="#2A7BDE"><b>--&gt; </b></font>/somewhere/expander/target/debug/build/expander-8cb9d7a52d4e83d1/out/baz-73b3d5b9bc4641a894d85b878020da6e331ecfd8949c9157c50a92845412a534.rs:2:42
  <font color="#2A7BDE"><b>|</b></font>
<font color="#2A7BDE"><b>2</b></font> <font color="#2A7BDE"><b>| </b></font>#[derive(Debug, Clone, Copy)] struct B ; ;
  <font color="#2A7BDE"><b>| </b></font>                                         <font color="#F66151"><b>^</b></font>

<font color="#C01C28"><b>error</b></font><b>:</b> could not compile `expander` due to 2 previous errors
<font color="#FFBF00"><b>warning</b></font><b>:</b> build failed, waiting for other jobs to finish...
<font color="#C01C28"><b>error</b></font><b>:</b> build failed
</pre>
---

which tells you exactly where in the generated code of your proc-macro you generated that superfluous
statement.

Now this was a simple example, doing this with macros that would expand to multiple tens of thousand lines of
code with `cargo-expand`, but only in a few thousand that your particular one generates, it's a
life saver to know what caused the issue rather than having to use `eprintln!` to print a unformated
string to the terminal.

> Hint: You can quickly toggle this by using `.dry(true || false)`