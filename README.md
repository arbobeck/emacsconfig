# emacsconfig

This is a public repository documenting my DOOM EMACS configuration (work in progress).

early-init.el --- Doom's universal bootstrapper -*- lexical-binding: t -*-
;;
;;
;;; Commentary:
;;
;;
;; This file, in summary:
;; - Determines where `user-emacs-directory' is by:
;;   - Processing `--init-directory DIR' (backported from Emacs 29),
;;   - Processing `--profile NAME' (see
;;     `https://docs.doomemacs.org/-/developers' or docs/developers.org),
;;   - Or assume that it's the directory this file lives in.
;; - Loads Doom as efficiently as possible, with only the essential startup
;;   optimizations, and prepares it for interactive or non-interactive sessions.
;; - If Doom isn't present, then we assume that Doom is being used as a
;;   bootloader and the user wants to load a non-Doom config, so we undo all our
;;   global side-effects, load `user-emacs-directory'/early-init.el, and carry
;;   on as normal (without Doom).
;; - Do all this without breaking compatibility with Chemacs.
;;
;; early-init.el was introduced in Emacs 27.1. It is loaded before init.el,
;; before Emacs initializes its UI or package.el, and before site files are
;; loaded. This is great place for startup optimizing, because only here can you
;; *prevent* things from loading, rather than turn them off after-the-fact.
;;
;; In Emacs 29, there is still early-init.el
;;
;;
;;
;; Doom uses this file as its "universal bootstrapper" for both interactive and
;; non-interactive sessions. That means: no matter what environment you want
;; Doom in, load this file first.
;;
;;; Code:



;; PERF: Garbage collection is a big contributor to startup times. This fends it
;;   off, but will be reset later by `gcmh-mode'. Not resetting it later will
;;   cause stuttering/freezes.
(setq gc-cons-threshold most-positive-fixnum)



;; PERF: Don't use precious startup time checking mtime on elisp bytecode.
;;   Ensuring correctness is 'doom sync's job, not the interactive session's.
;;   Still, stale byte-code will cause *heavy* losses in startup efficiency.
(setq load-prefer-newer noninteractive)



;; UX: Respect DEBUG envvar as an alternative to --debug-init, and to make are
;;   startup sufficiently verbose from this point on.
(when (getenv-internal "DEBUG")
  (setq init-file-debug t
        debug-on-error t))


;;
;;; Bootstrap

(or
 ;; PERF: `file-name-handler-alist' is consulted often. Unsetting it offers a
 ;;   notable saving in startup time. This let-binding is just a stopgap though,
 ;;   a more complete version of this optimization can be found in lisp/doom.el.
 (let (file-name-handler-alist)
   (let* (;; FIX: Unset `command-line-args' in noninteractive sessions, to
          ;;   ensure upstream switches aren't misinterpreted.
          (command-line-args (unless noninteractive command-line-args))
          ;; I avoid using `command-switch-alist' to process --profile (and
          ;; --init-directory) because it is processed too late to change
          ;; `user-emacs-directory' in time.
          (profile (or (cadr (member "--profile" command-line-args))
                       (getenv-internal "DOOMPROFILE"))))
     (if (null profile)
         ;; REVIEW: Backported from Emacs 29. Remove when 28 support is dropped.
         (let ((init-dir (or (cadr (member "--init-directory" command-line-args))
                             (getenv-internal "EMACSDIR"))))
           (if (null init-dir)
               ;; FIX: If we've been loaded directly (via 'emacs -batch -l
               ;;   early-init.el') or by a doomscript (like bin/doom), and Doom
               ;;   is in a non-standard location (and/or Chemacs is used), then
               ;;   `user-emacs-directory' will be wrong.
               (when noninteractive
                 (setq user-emacs-directory
                       (file-name-directory (file-truename load-file-name))))
             ;; FIX: To prevent "invalid option" errors later.
             (push (cons "--init-directory" (lambda (_) (pop argv))) command-switch-alist)
             (setq user-emacs-directory (expand-file-name init-dir))))
       ;; FIX: Discard the switch to prevent "invalid option" errors later.
       (push (cons "--profile" (lambda (_) (pop argv))) command-switch-alist)
       ;; Running 'doom sync' or 'doom profile sync' (re)generates a light
       ;; profile loader in $EMACSDIR/profiles/load.el (or
       ;; $DOOMPROFILELOADFILE), after reading `doom-profile-load-path'. This
       ;; loader requires `$DOOMPROFILE' be set to function.
       (setenv "DOOMPROFILE" profile)
       (or (load (expand-file-name
                  (format (let ((lfile (getenv-internal "DOOMPROFILELOADFILE")))
                            (if lfile
                                (concat (string-remove-suffix ".el" lfile)
                                        ".%d.elc")
                              "profiles/load.%d.elc"))
                          emacs-major-version)
                  user-emacs-directory)
                 'noerror (not init-file-debug) 'nosuffix)
           (user-error "Profiles not initialized yet; run 'doom sync' first"))))

   ;; PERF: When `load'ing or `require'ing files, each permutation of
   ;;   `load-suffixes' and `load-file-rep-suffixes' (then `load-suffixes' +
   ;;   `load-file-rep-suffixes') is used to locate the file.  Each permutation
   ;;   is a file op, which is normally very fast, but they can add up over the
   ;;   hundreds/thousands of files Emacs needs to load.
   ;;
   ;;   To reduce that burden -- and since Doom doesn't load any dynamic modules
   ;;   -- I remove `.so' from `load-suffixes' and pass the `must-suffix' arg to
   ;;   `load'. See the docs of `load' for details.
   (if (let ((load-suffixes '(".elc" ".el")))
         ;; I avoid `load's NOERROR argument because other, legitimate errors
         ;; (like permission or IO errors) should not be suppressed or
         ;; interpreted as "this is not a Doom config".
         (condition-case _
             ;; Load the heart of Doom Emacs.
             (load (expand-file-name "lisp/doom" user-emacs-directory)
                   nil (not init-file-debug) nil 'must-suffix)
           ;; Failing that, assume that we're loading a non-Doom config.
           (file-missing
            ;; HACK: `startup--load-user-init-file' resolves $EMACSDIR from a
            ;;   lexically bound `startup-init-directory', which means changes
            ;;   to `user-emacs-directory' won't be respected when loading
            ;;   $EMACSDIR/init.el, so I force it to:
            (define-advice startup--load-user-init-file (:filter-args (args) reroute-to-profile)
              (list (lambda () (expand-file-name "init.el" user-emacs-directory))
                    nil (nth 2 args)))
            ;; Set `user-init-file' for the `load' call further below, and do so
            ;; here while our `file-name-handler-alist' optimization is still
            ;; effective (benefits `expand-file-name'). BTW: Emacs resets
            ;; `user-init-file' and `early-init-file' after this file is loaded.
            (setq user-init-file (expand-file-name "early-init" user-emacs-directory))
            ;; COMPAT: I make no assumptions about the config we're going to
            ;;   load, so undo this file's global side-effects.
            (setq load-prefer-newer t)
            ;; PERF: But make an exception for `gc-cons-threshold', which I
            ;;   think all Emacs users and configs will benefit from. Still,
            ;;   setting it to `most-positive-fixnum' is dangerous if downstream
            ;;   does not reset it later to something reasonable, so I use 16mb
            ;;   as a best fit guess. It's better than Emacs' 80kb default.
            (setq gc-cons-threshold (* 16 1024 1024))
            nil)))
       ;; ...But if Doom loaded then continue as normal.
       (doom-require (if noninteractive 'doom-cli 'doom-start))))










.el -*- lexical-binding: t; -*-

;; This file controls what Doom modules are enabled and what order they load
;; in. Remember to run 'doom sync' after modifying it!

;; NOTE Press 'SPC h d h' (or 'C-h d h' for non-vim users) to access Doom's
;;      documentation. There you'll find a link to Doom's Module Index where all
;;      of our modules are listed, including what flags they support.

;; NOTE Move your cursor over a module's name (or its flags) and press 'K' (or
;;      'C-c c k' for non-vim users) to view its documentation. This works on
;;      flags as well (those symbols that start with a plus).
;;
;;      Alternatively, press 'gd' (or 'C-c c d') on a module to browse its
;;      directory (for easy access to its source code).

(doom! :input
       ;;bidi              ; (tfel ot) thgir etirw uoy gnipleh
       ;;chinese
       ;;japanese
       ;;layout            ; auie,ctsrnm is the superior home row

       :completion
       company           ; the ultimate code completion backend
       ;;helm              ; the *other* search engine for love and life
       ;;ido               ; the other *other* search engine...
       ivy               ; a search engine for love and life
       vertico           ; the search engine of the future

       :ui
       ;;deft              ; notational velocity for Emacs
       doom              ; what makes DOOM look the way it does
       doom-dashboard    ; a nifty splash screen for Emacs
       ;;doom-quit         ; DOOM quit-message prompts when you quit Emacs
       ;;(emoji +unicode)  ; 🙂
       hl-todo           ; highlight TODO/FIXME/NOTE/DEPRECATED/HACK/REVIEW
       ;;hydra
       indent-guides     ; highlighted indent columns
       ligatures         ; ligatures and symbols to make your code pretty again
       ;;minimap           ; show a map of the code on the side
       modeline          ; snazzy, Atom-inspired modeline, plus API
       ;;nav-flash         ; blink cursor line after big motions
       ;;neotree           ; a project drawer, like NERDTree for vim
       ophints           ; highlight the region an operation acts on
       (popup +defaults)   ; tame sudden yet inevitable temporary windows
       ;;tabs              ; a tab bar for Emacs
       treemacs          ; a project drawer, like neotree but cooler
       ;;unicode           ; extended unicode support for various languages
       (vc-gutter +pretty) ; vcs diff in the fringe
       vi-tilde-fringe   ; fringe tildes to mark beyond EOB
       ;;window-select     ; visually switch windows
       workspaces        ; tab emulation, persistence & separate workspaces
       ;;zen               ; distraction-free coding or writing

       :editor
       (evil +everywhere); come to the dark side, we have cookies
       file-templates    ; auto-snippets for empty files
       fold              ; (nigh) universal code folding
       ;;(format +onsave)  ; automated prettiness
       ;;god               ; run Emacs commands without modifier keys
       ;;lispy             ; vim for lisp, for people who don't like vim
       ;;multiple-cursors  ; editing in many places at once
       ;;objed             ; text object editing for the innocent
       ;;parinfer          ; turn lisp into python, sort of
       ;;rotate-text       ; cycle region at point between text candidates
       snippets          ; my elves. They type so I don't have to
       ;;word-wrap         ; soft wrapping with language-aware indent

       :emacs
       dired             ; making dired pretty [functional]
       electric          ; smarter, keyword-based electric-indent
       ;;ibuffer         ; interactive buffer management
       undo              ; persistent, smarter undo for your inevitable mistakes
       vc                ; version-control and Emacs, sitting in a tree

       :term
       eshell            ; the elisp shell that works everywhere
       ;;shell             ; simple shell REPL for Emacs
       term              ; basic terminal emulator for Emacs
       vterm             ; the best terminal emulation in Emacs

       :checkers
       syntax              ; tasing you for every semicolon you forget
       ;;(spell +flyspell) ; tasing you for misspelling mispelling
       ;;grammar           ; tasing grammar mistake every you make

       :tools
       ;;ansible
       ;;biblio            ; Writes a PhD for you (citation needed)
       ;;debugger          ; FIXME stepping through code, to help you add bugs
       ;;direnv
       ;;docker
       ;;editorconfig      ; let someone else argue about tabs vs spaces
       ;;ein               ; tame Jupyter notebooks with emacs
       (eval +overlay)     ; run code, run (also, repls)
       ;;gist              ; interacting with github gists
       lookup              ; navigate your code and its documentation
       ;;lsp               ; M-x vscode
       magit             ; a git porcelain for Emacs
       ;;make              ; run make tasks from Emacs
       ;;pass              ; password manager for nerds
       ;;pdf               ; pdf enhancements
       ;;prodigy           ; FIXME managing external services & code builders
       ;;rgb               ; creating color strings
       ;;taskrunner        ; taskrunner for all your projects
       ;;terraform         ; infrastructure as code
       ;;tmux              ; an API for interacting with tmux
       ;;tree-sitter       ; syntax and parsing, sitting in a tree...
       ;;upload            ; map local to remote projects via ssh/ftp

       :os
       (:if IS-MAC macos)  ; improve compatibility with macOS
       ;;tty               ; improve the terminal Emacs experience

       :lang
       ;;agda              ; types of types of types of types...
       ;;beancount         ; mind the GAAP
       ;;(cc +lsp)         ; C > C++ == 1
       ;;clojure           ; java with a lisp
       ;;common-lisp       ; if you've seen one lisp, you've seen them all
       ;;coq               ; proofs-as-programs
       ;;crystal           ; ruby at the speed of c
       ;;csharp            ; unity, .NET, and mono shenanigans
       ;;data              ; config/data formats
       ;;(dart +flutter)   ; paint ui and not much else
       ;;dhall
       ;;elixir            ; erlang done right
       ;;elm               ; care for a cup of TEA?
       emacs-lisp        ; drown in parentheses
       ;;erlang            ; an elegant language for a more civilized age
       ;;ess               ; emacs speaks statistics
       ;;factor
       ;;faust             ; dsp, but you get to keep your soul
       ;;fortran           ; in FORTRAN, GOD is REAL (unless declared INTEGER)
       ;;fsharp            ; ML stands for Microsoft's Language
       ;;fstar             ; (dependent) types and (monadic) effects and Z3
       ;;gdscript          ; the language you waited for
       ;;(go +lsp)         ; the hipster dialect
       ;;(graphql +lsp)    ; Give queries a REST
       ;;(haskell +lsp)    ; a language that's lazier than I am
       ;;hy                ; readability of scheme w/ speed of python
       ;;idris             ; a language you can depend on
       json              ; At least it ain't XML
       ;;(java +lsp)       ; the poster child for carpal tunnel syndrome
       javascript        ; all(hope(abandon(ye(who(enter(here))))))
       ;;julia             ; a better, faster MATLAB
       ;;kotlin            ; a better, slicker Java(Script)
       ;;latex             ; writing papers in Emacs has never been so fun
       ;;lean              ; for folks with too much to prove
       ;;ledger            ; be audit you can be
       ;;lua               ; one-based indices? one-based indices
       markdown          ; writing docs for people to ignore
       ;;nim               ; python + lisp at the speed of c
       ;;nix               ; I hereby declare "nix geht mehr!"
       ;;ocaml             ; an objective camel
       org               ; organize your plain life in plain text
       ;;php               ; perl's insecure younger brother
       ;;plantuml          ; diagrams for confusing people more
       ;;purescript        ; javascript, but functional
       python            ; beautiful is better than ugly
       ;;qt                ; the 'cutest' gui framework ever
       ;;racket            ; a DSL for DSLs
       ;;raku              ; the artist formerly known as perl6
       ;;rest              ; Emacs as a REST client
       ;;rst               ; ReST in peace
       ;;(ruby +rails)     ; 1.step {|i| p "Ruby is #{i.even? ? 'love' : 'life'}"}
       ;;(rust +lsp)       ; Fe2O3.unwrap().unwrap().unwrap().unwrap()
       ;;scala             ; java, but good
       ;;(scheme +guile)   ; a fully conniving family of lisps
       sh                ; she sells {ba,z,fi}sh shells on the C xor
       ;;sml
       ;;solidity          ; do you need a blockchain? No.
       ;;swift             ; who asked for emoji variables?
       ;;terra             ; Earth and Moon in alignment for performance.
       web               ; the tubes
       ;;yaml              ; JSON, but readable
       ;;zig               ; C, but simpler

       :email
       ;;(mu4e +org +gmail)
       ;;notmuch
       ;;(wanderlust +gmail)

       :app
       ;;calendar
       ;;emms
       ;;everywhere        ; *leave* Emacs!? You must be joking
       ;;irc               ; how neckbeards socialize
       ;;(rss +org)        ; emacs as an RSS reader
       ;;twitter           ; twitter client https://twitter.com/vnought

       :config
       ;;literate
       (default +bindings +smartparens))

 (load user-init-file 'noerror (not init-file-debug) nil 'must-suffix))

;;; early-init.el ends here

Set up gmail in emacs.
