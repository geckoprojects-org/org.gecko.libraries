This repackaging has been made to fix [this issue](https://github.com/google/conscrypt/issues/1173). The dependency from `sun.security.x509` is not actually used, because it is only used in deprecated code. 

Thus, we repackaged the library adding an **optional** import for that dependency.