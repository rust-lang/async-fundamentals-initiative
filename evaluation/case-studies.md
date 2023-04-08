# ⚖️ Case Studies

*Spring 2023:* In preparation for stabilization an MVP of async function in traits, we asked a number of teams to integrate the nightly support on an experimental basis and talk through how well it worked for them. The following case studies provide details on their experiences. In short, the conclusions were:

* Static dispatch AFIT generally meets expectations.
* Most every project needs dynamic dispatch, but the workaround of deriving an "erased" version of the trait is sufficient for now, and can be hidden from end-users as a private impl detail.
* Some solution for applying send bounds is required. RTN is fine for most cases but doesn't scale well to traits with many methods.
* Having some automated way to derive "dynamic compatible" and "all methods with send bound" variants of the trait would be useful.