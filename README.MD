# Scrappy #

# <img src="./scrappy-logo.png.png" width="200">


provides a number of functions that allow for easily scraping certain patterns from websites as well control functions that allow you to rotate between multiple sites and rotate proxies in order to deal with common bot detection problems faced by scrapers. 

## Scraping ## 

Scraping is parsing, where we don't care about the placement of our match in a chunk of data. ScraperT is really just ParsecT (ParsecT String () Identity a)

## How to depend on this project ## 

Scrappy is currently not on Hackage because I honestly don't have time to perfect version bounds however through [Nix](https://nixos.org/download) you can get the package the following way:

OR!! Just clone scrappy-tutorial and rename it 

OR if you really want to learn (hey good for you!) 

```bash 
# assuming you've done nix-setup... I'd show you how but it depends what setup you prefer and its super simple
nix-env -f https://github.com/obsidiansystems/nix-thunk/archive/master.tar.gz -iA command
nix-shell -p cabal2nix
cd yourProjectDirectory
cabal init 
# follow prompts
mkdir deps 
git clone https://github.com/Ace-Interview-Prep/scrappy.git deps/scrappy # or SSH url 
nix-thunk pack deps/scrappy # this is technically optional but recommended
# add scrappy to your cabal file under build-depends
cabal2nix . --shell > shell.nix -- this will create an easy to use shell.nix file with all of your build depends
nix-shell #you could also do nix-shell shell.nix
cabal run my-project-name-in-cabal-file 
```


## Tutorials ## 

- https://medium.com/p/135283dc2af
- https://github.com/Ace-Interview-Prep/scrappy-tutorial

## Main functions, Types, and Classes ## 

```haskell

class ElementRep (a :: * -> *) where
--type InnerHTMLRep a = something 
  elTag :: a b -> Elem
  attrs :: a b -> Attrs
  innerText' :: a b -> String 
  matches' :: a b -> [b]

instance ElementRep (Elem') where
  elTag = _el
  attrs = _attrs
  innerText' = innerHtmlFull
  matches' = innerMatches

data Elem' a = Elem' { _el :: Elem 
                     , _attrs :: Map String String 
                     , innerMatches :: [a] 
                     , innerHtmlFull :: String
                     } deriving Show

-- | Manager returned may be a new manager, if the given manager failed 
getHtml :: Manager -> Url -> IO (Manager, Html) 
getHtml' :: Url -> IO Html

scrape :: ScraperT a -> Html -> Maybe [a]

-- | For more advanced control over inner structure see Elem.TreeElemParser
elemParser :: Elem -> [(String, Maybe String)] 
            -> Maybe (ParsecT s u m a) -- Optionally scraped pattern inside this el, if specified, return element must have at least 1 
            -> ScraperT (Elem' a)

el :: Elem -> [(String, String)] -> ScraperT (Elem' String)

-- | Find any expressive pattern as long as it is inside of some HTML context 
contains :: ScraperT (Elem' a) -> ScraperT a -> ScraperT a 


:t fmap snd getHtml' "https://google.com" >>= return . (runScraperOnHtml (el "a" [])   
>>> IO (Maybe [Elem' a])

-- Will get all <a id="link"> 's on the url that are inside
-- divs with a class of xyz
example :: IO (Maybe [Elem' a])
example = runScraperOnUrl url $ el "div" [("class", "xyz")] `contains`
        el "a" [("id", "link")]


```

Currently with this library you can request HTML and run a multitude of scraper-patterns on it, for example:
- Scrape an element
- Scrape an element with a specific inner pattern
- Scrape an element with an attribute that fits a specific parser-pattern 
- Scrape an element with a specific inner DOM-tree 
- Scrape a group of elements that *kinda* repeats 
   - For example, if you want a complex group that varies from page to page but to the user's eye looks the exact same (think of search results on a nice website) or even largely the same, then use the `htmlGroup` function from Scrappy.Elem.TreeElemParser  
- Scrape any **arbitrary** parser from the [parsec library](https://hackage.haskell.org/package/parsec)

You can also deeply chain! 
- Inside any instance of the ElementRep class (Elem', TreeHTML) through contains and contains'! 
   - For example use: think of getting all prices 
- In a sequence!
   - skipToBody: to avoid matches in the <head>
   - (</>>=) and (</>>) which are sequencers that take two parsers like Monadic actions (hence the characters chosen) but unlike running two parsers one after the other, there may be whatever random stuff in between! 


## TODO ## 

As this is an Open Source ambitious project, there is much that is left to do which follows from the potential of this library as a result of it's inner workings

- Upload to Hackage
- Be included in the haskellPackages attribute of nixpkgs
- grep/regex replacement (and streamEdit across multiple files, see below) 
  - I hate ReGeX with a burning passion. No way will I ever come back to an expression and say "Oh that must match on emails!". 
  - It would be neat to have a way to write haskell at the command line like "grepScrappy -sR "char 'a' >> some digit >> char 'b'" but this string would quickly get massive so I wonder if there's a happy medium that exists such as shortforms for the most common parsers (char, many, some, digit, string, etc) that while we would still allow for any haskell string (which typifies to a ParsecT) there would be ways to shorten expressions
- Improve `contains` and similar functions to be able to somehow apply the 'contained' parser to inside the start and end tags
- More Generic typing and Monads for a SiteT, HistoryT, and WebT (an accumulator of new undiscovered sites)
   - I have a number of complicated Monads all with their own varied implementations that could be greatly reduced to a SiteT which keeps state of what branches of the Site tree have been visited or seen.
      - With a SiteT it would easily allow for logic to gather all new links (see Scrappy.Links) upon getting a new HTML via getHTML or sibling functions, and perhaps a new sibling function like getNextHTML which gets the next untouched HTML page 
      - SiteT extension would mean that we could create functions like `scrapeSite` which might look for all pattern matches across the site-tree. By extension this could apply to HistoryT and WebT via an incredibly simple interface. 
   - The concept of a SiteT implies there may be use in a HistoryT (ie all Sites visited with their respective site-trees) and a WebT which is meant to represent the entire web in the sense that it continues to extend and is not constrained to past history.
   - Hopefully this gives intriguing ideas to why scrappy-nlp might be so powerful
- Concurrency with streamEdits 
   - This functionality will help both for making it easier+quicker to perform concurrency when scraping (which is actually quite easy for different pages due to forkIO and standard library functions) but also makes a scrappy-DOM (see below) much easier to imagine as a robust web framework (imagine how long 10+ prerender-like scrapers would take or rather 10 passthroughs). I see this as the biggest blocker to a 1st version of scrappy-DOM (which wouldn't yet have FRP or GHCJS functionality.. you could still write raw JavaScript in V1 though) 
   - Extending this idea, lazy-js (see below) is effectively a streamEdit performer and so this logic would also provide incredible speed to this 'category' of the DOM as well. 
- Break up into scrappy-core and scrappy-requests
   - This is so that scrappy-core can be used in frontend apps that are built using GHCJS. GHCJS purposely doesn't build networking packages. 
- Fork a streamEdit package
   - A lot of streamEdit packages are exceptionally complex while library functions like Data.Text leave much to be desired. scrappy-streamEdit would be a perfect medium
- Fork a scrappy-staleLink (or better name) package 
   - For use with [Obelisk](https://github.com/obsidiansystems/obelisk) the static directory which will provide a list of static assets which are no longer in use and might be good to delete
- Fork a scrappy-templates package
   - This is in the works currently for templating files/strings with variables such as large prompts for GPT-*, for example you could template a prompt for resumes that is job dependent in an ergonomic/easy-to-use way
   - See Scrappy.Templates
   - To reduce runtime errors from performing IO, it would be nice to have a staticFile finder like with staticWhich: a package that ensures an executable's existence *at compile time* 
- scrappy + nlp
   - It would be advantageous to use certain functions like a recursive scraper from pageN -> page(N+M), where M is based on a stop condition and/or the arrow a successful pattern match (see Scrappy.Run). While this can be easily done with HTML patterns, this extends and lends well to NLP analysis for the text that is contained in the structural HTML
- scrappy-DOM
   - I currently have a bone to pick with the `prerender` function from [reflex-dom](https://github.com/reflex-frp/reflex-dom) as it can easily fail. For example, if the browser insists on inserting a <tbody> inside a <table> tag then reflex-dom panics because the DOM is not as expected, even if the DOM area is on a completely different branch and way earlier than it. Intuitively, using streamEdit on an element with an attribute like prerenderID="some unique string" would be an incredibly dependable solution. 
   - The blockers to this are of course that it's only worth doing if it will be an improvement to reflex-dom as a whole (or maybe we integrate with reflex-dom) and reflex-dom is truly exceptional with FRP, GHCJS bindings, and mobile-readiness 
- scrappy-headless
   - This is effectively a headless browser which has the ability to read and run JS in response to events. A lot of work has been done on this in the Scrappy.BuildActions module and in [lazy-js](https://github.com/augyg/lazy-js) which is a fork of this project
   - Also need to implement httpHistory in a more robust way to make a headless browser more legit 
- lazy-js 
   - This is a foundational project for both scrappy-headless and scrappy-DOM which is a lazy reader and writer of JavaScript. An example use is virtualizing clicking a link to bypass any JavaScript bot detectors and/or just to follow JavaScript based events like a button which calls some JS to change window.location (like a fancy href)

