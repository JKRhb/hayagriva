# Hayagriva

[![Build status](https://github.com/typst/hayagriva/workflows/Continuous%20integration/badge.svg)](https://github.com/typst/hayagriva/actions)

Rusty bibliography management.

Hayagriva is a tool that can help your or your apps deal with literature and
other media. Its features include:

- Data structures for literature collections
- Reading and writing said collections from YAML files
- Formatting literature into reference list entries and in-text citations as
  defined by popular style guides
- Interoperability with BibTeX
- Querying your literature items by type and available metadata

Hayagriva can be used both as a library and as a Command Line Interface (CLI).
Skip to the [section "Usage"](#usage) for more information about usage in your
application or to the [section "Installation"](#installation) to learn about how
to install and use Hayagriva on your terminal.

## Supported styles

- Institute of Electrical and Electronics Engineers (IEEE)
    - References
    - Numerical citations
- Modern Language Association (MLA), 8th edition of the MLA Handbook
    - "Works Cited" references
- Chicago Manual of Style (CMoS), 17th edition
    - Notes and Bibliography
    - Author-Date references and citations
- American Psychological Association (APA), 7th edition of the APA Publication Manual
    - References
- Other in-text citation styles
    - Alphanumerical (e. g. "Rass97")
    - Author Title

## Usage

Add this to your `Cargo.toml`:
```toml
[dependencies]
hayagriva = "0.1"
```

Below, there is an example of how to parse a YAML database and get a Modern
Language Association-style citation.

```rust
use hayagriva::io::from_yaml_str;
use hayagriva::style::{Database, Mla};

let yaml = r#"
crazy-rich:
    type: Book
    title: Crazy Rich Asians
    author: Kwan, Kevin
    date: 2014
    publisher: Anchor Books
    location: New York, NY, US
"#;

// Parse a bibliography
let bib = from_yaml_str(yaml).unwrap();
assert_eq!(bib[0].date().unwrap().year, 2014);

// Format the reference
let db = Database::from_entries(bib.iter());
let mut mla = Mla::new();
let reference = db.bibliography(&mut mla, None);
assert_eq!(reference[0].display.value, "Kwan, Kevin. Crazy Rich Asians. Anchor Books, 2014.");
```

Formatting for in-text citations is available through implementors of the
`hayagriva::style::CitationStyle` trait whereas bibliographies can be created by
`hayagriva::style::BibliographyStyle`. Both traits are used through
`style::Database` which provides methods to format its records as bibliographies
and citations.

If the default features are enabled, Hayagriva supports BibTeX and BibLaTeX
bibliographies. You can use `io::from_biblatex_str` to parse such
bibliographies.

Should you need more manual control, the library's native `Entry` struct also
offers an implementation of the `From<&biblatex::Entry>`-Trait. You will need to
depend on the [biblatex](https://docs.rs/biblatex/latest/biblatex/) crate to
obtain its `Entry`. Therefore, you could also use your BibLaTeX content like
this:

```rust
use hayagriva::Entry;
let converted: Entry = your_biblatex_entry.into();
```

If you do not need BibLaTeX compatibility, you can use Hayagriva without the
default features by writing this in your `Cargo.toml`:

```toml
[dependencies]
hayagriva = { version = "0.1", default-features = false }
```

### Selectors

Hayagriva uses a custom selector language that enables you to filter
bibliographies by type of media. For more information about selectors, refer to
the [selectors.md
file](https://github.com/typst/hayagriva/blob/main/docs/selectors.md). While you
can parse user-defined selectors using the function `Selector::parse`, you may
instead want to use the selector macro to avoid the run time cost of parsing a
selector when working with constant selectors.

```rust
use hayagriva::select;
use hayagriva::io::from_yaml_str;

let yaml = r#"
quantized-vortex:
    type: Article
    author: Gross, E. P.
    title: Structure of a Quantized Vortex in Boson Systems
    date: 1961-05
    page-range: 454-477
    doi: 10.1007/BF02731494
    parent:
        issue: 3
        volume: 20
        title: Il Nuovo Cimento
"#;

let entries = from_yaml_str(yaml).unwrap();
let journal = select!((Article["date"]) > ("journal":Periodical));
assert!(journal.matches(&entries[0]));
```

There are two ways to check if a selector matches an entry.
You should use [`Selector::matches`] if you just want to know if an item
matches a selector and [`Selector::apply`] to continue to work with the data from
parents of a matching entry. Keep in mind that the latter function will
return `Some` even if no sub-entry was bound / if the hash map is empty.

## Installation

Run this in your terminal:
```bash
cargo install hayagriva --features cli
```

Cargo will install the Hayagriva Command Line Interface for you. Now, you just
need a Hayagriva YAML literature file or a Bib(La)TeX file to get started. The
Hayagriva YAML file is intuitive to write and can represent a wealth of media
types, [learn how to write one in its dedicated
documentation.](https://github.com/typst/hayagriva/blob/main/docs/file-format.md)

Suppose you have this file saved as `literature.yml` in your current working
directory:

```yaml
dependence:
    type: Article
    title: The program dependence graph and its use in optimization
    author: ["Ferrante, Jeanne", "Ottenstein, Karl J.", "Warren, Joe D."]
    date: 1987-07
    doi: "10.1145/24039.24041"
    parent:
        type: Periodical
        title: ACM Transactions on Programming Languages and Systems
        volume: 9
        issue: 3

feminism:
    type: Article
    title: She swoons to conquer
    author: Ungard-Sargon, Batya
    editor: Weintraub, Pam
    date: 2015-09-25
    url: https://aeon.co/essays/can-you-enjoy-romance-fiction-and-be-a-feminist
    parent:
        type: Blog
        title: Aeon
```

You can then issue the following command to get reference list entries for both
of these articles.
```bash
hayagriva literature.yml reference
```

Hayagriva defaults to the Author-Date style of the Chicago Manual of Style (17th
edition). If you prefer to use another style, you can, for example, do the
following to use the style of the American Psychological Association instead:
```bash
hayagriva literature.yml reference --style apa
```

Available values for the `--style` argument@ can be viewed by calling
`hayagriva help reference`.

If you now need an in-text citation to the second article in the above file, you
can call:
```bash
hayagriva literature.yml --key feminism citation
```

The `--key` argument precedes the sub-command (e. g. `citation`, `reference`)
and takes a comma-separated list of keys (or a single one). The sub-command will
then only work on the specified keys. Just like the `reference` sub-command, the
`citation` command also allows for the `--style` argument. Its possible values
can be viewed with `hayagriva help citation`. It will default to the _Author Date_
style.

Instead of the `--key` argument, you can also use `--select` to provide a custom
[Hayagriva selector.](https://github.com/typst/hayagriva/blob/main/docs/selectors.md)
For example, you could run the following to only reference entries that have a URL
or DOI at the top level:
```bash
hayagriva literature.yml --select "*[url] | *[doi]" reference
```

This expression would match both entries in our example and therefore the
command would return the same result as the first reference command.

Hayagriva also allows you to explore which values were bound to which
sub-entries if the selector matches. This is especially useful if you intend to
consume Hayagriva as a dependency in your application and need to debug an
expression. Consider this selector which always binds the sub-entry with the
volume field to `a`, regardless of if it occurred at the top level or in the
first parent: `a:*[volume] | * > a:[volume]`. You can then use the command below
to show which sub-entry the selector bound as `a` for each match:
```bash
hayagriva literature.yml --select "a:*[volume] | * > a:[volume]" keys --show-bound
```

The `keys` sub-command enumerates all keys that match the selector / key
conditions that you specified or all keys in the bibliography if there are none.
The `--show-bound` flag then shows the selector bindings for each match.

If you are working with BibTeX, you can use your `.bib` file with Hayagriva just
like you would use a `.yml` file. If you want to convert your `.bib` file to a
`.yml` file, you can run this, which will save the conversion result in the
current working directory as `converted.yml`:
```bash
hayagriva literature.bib dump -o converted.yml
```

The converted file will instead be printed to the terminal if you omit the `-o`
argument.

## Contributing

We are looking forward to receiving your bugs and feature requests in the Issues
tab. We would also be very happy to accept PRs for bug fixes, minor
refactorings, features that were requested in the issues and greenlit by us, as
well as the planned features listed below:

- More citation and reference styles (especially styles used in the 'hard'
  sciences would be incredibly appreciated)
- Implementing the YAML-to-BibLaTeX conversion
- Improvements to the sentence and title formatter
- Work for non-English bibliographies
- Documentation improvements

We wish to thank each and every prospective contributor for the effort you (plan
to) invest in this project and for adopting it!

## License

Hayagriva is licensed under a MIT / Apache 2.0 dual license.

Users and consumers of the library may choose which of those licenses they want
to apply whereas contributors have to accept that their code is in compliance
and distributed under the terms of both of these licenses.
