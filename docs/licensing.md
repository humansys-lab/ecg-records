# Licensing

`ecg-records` is distributed as two PyPI packages with different licences.

| Package | What it contains | Licence | Form |
|---|---|---|---|
| `ecg-records` | the ECG Record format, readers, queries, plots, the CLI and the measurement pipeline *shell* | [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0) | pure Python |
| `ecg-records-engine` | the numerical engine: QRS detection, P/QRS/T delineation, intervals, amplitudes, axes, ST | **PolyForm Noncommercial 1.0.0** for noncommercial use, a **commercial licence** otherwise | compiled wheels (no Python source) |

`pip install "ecg-records[engine]"` installs both, at the same minor version. Reading,
querying and plotting ECG Records needs only `ecg-records`; measuring a signal
(`ecg_record`, `ecg_measure`, `ecg-record measure`) needs the engine.

## Who can use the engine for free

The [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0)
permits any **noncommercial purpose**:

- personal use (research, study, hobby);
- use by educational institutions, charities, government and public research
  organizations, and other organizations that do not operate for profit.

Everything else, in particular use **by or for a company**, use in a product
or service, and paid work for a client, needs a commercial licence. Academic
research inside a university is free; the same code running inside a hospital
group that operates for profit, or inside a device company's product, is not.
The licence text is the authority; this page is a summary.

## Commercial licence

A commercial licence is a written agreement with humansys-lab (the standard
terms ship with the engine as `LICENSE-COMMERCIAL.md`) plus a **licence key**.
Licences are sold per site or per seat for a term; contact the maintainers
through the repository's issue tracker or the address in the package metadata.

### Installing a key

The key is a single-line token starting with `ECGR1.`. Store it in one of
these places (first match wins):

1. the environment variable `ECG_RECORDS_LICENSE_KEY` (the token itself);
2. the file named by the environment variable `ECG_RECORDS_LICENSE`;
3. `~/.config/ecg-records/license.key` on Linux/macOS
   (`$XDG_CONFIG_HOME/ecg-records/license.key` if set), or
   `%APPDATA%\ecg-records\license.key` on Windows.

Check it:

```python
import ecgfeat
ecgfeat.license_info()
# {'status': 'commercial', 'licensee': 'Example GmbH', 'expires': '2027-10-15', ...}
```

`status` is `noncommercial` (no key found), `commercial`, `expired`, `invalid`
(a key was found but does not verify; the reason is given) or
`engine_not_installed`.

### What the key does, and does not do

- Verification is **offline**: the key carries an Ed25519 signature that the
  engine checks against a public key compiled into it. Nothing is sent
  anywhere; the engine works on air-gapped machines.
- The key does not unlock features. Without a valid commercial key the engine
  prints **one notice per process** on first use (to `stderr`) and otherwise
  behaves identically.
- Keys are bound to the engine versions released during the licence term and
  keep working with those versions after it ends.

## Source code

The engine's source code is not published. The ECG Record schema, the
validation registry, the documentation and the examples are public, and the
`ecg-records` package is open source under Apache-2.0, so records produced by
the engine can always be read, verified and plotted with open code.
