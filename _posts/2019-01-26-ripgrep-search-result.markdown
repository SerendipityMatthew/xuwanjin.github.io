---
layout:     post
title:      "riggrep 搜索结果"
subtitle:   "riggrep 搜索结果"
date:       2019-01-25 20:30:40
author:     "Mathew"
catalog: true
header-img: "img/post-bg-2017.jpg"
tags:
    - rust
    - ripgrep
---


# riggrep 搜索结果

## 问题
![ripgrep](/img/apps/ripgrep/ripgrep_search_result.png)
最近发现riggrep，执行搜索命令的时候出现了结果不对的情况


## 思考
于是觉得这是一个bug， 只是很奇怪，rust 遍历系统当前的文件，然后唯独失去了```common```文件夹的搜索, 这事为什么呢?
当我 cd 到这个目录再执行的时候，他却能搜的出来结果, 很奇怪。

## 解决过程
### 查看源码
这个时候第一想法，是看看源代码，源码是怎么写的。这可能和我的一直做 Android Framework 的工作原因, 第一个想到的事情就是阅读源码. 
找了好久才找到了 ripgrep 的程序入口 ```ripgrep/src/main.rs```   
```rust
fn main() {
//这里是对命令行参数的解析
    match Args::parse().and_then(try_main) {
        Ok(true) => process::exit(0),
        Ok(false) => process::exit(1),
        Err(err) => {
            eprintln!("{}", err);
            process::exit(2);
        }
    }
}
```

```rust
fn try_main(args: Args) -> Result<bool> {
    use args::Command::*;

    match args.command()? {
        Search => search(args), //单线程搜索， 我主要看这里
        SearchParallel => search_parallel(args), //多线程搜索的
        SearchNever => Ok(false),
        Files => files(args),
        FilesParallel => files_parallel(args),
        Types => types(args),
    }
}
```

```rust
/// The top-level entry point for single-threaded search. This recursively
/// steps through the file list (current directory by default) and searches
/// each file sequentially.
fn search(args: Args) -> Result<bool> {
    let started_at = Instant::now();
    let quit_after_match = args.quit_after_match()?;
    let subject_builder = args.subject_builder();
    let mut stats = args.stats()?;
    let mut searcher = args.search_worker(args.stdout())?;
    let mut matched = false;

    for result in args.walker()? { //应该是这个地方开始遍历文件的，我想在这里对所有的路径加个log
        let subject = match subject_builder.build_from_result(result) {
            Some(subject) => subject,
            None => continue,
        };
        let search_result = match searcher.search(&subject) {
            Ok(search_result) => search_result,
            Err(err) => {
                // A broken pipe means graceful termination.
                if err.kind() == io::ErrorKind::BrokenPipe {
                    break;
                }
                message!("{}: {}", subject.path().display(), err);
                continue;
            }
        };
        matched = matched || search_result.has_match();
        if let Some(ref mut stats) = stats {
            *stats += search_result.stats().unwrap();
        }
        if matched && quit_after_match {
            break;
        }
    }
    if let Some(ref stats) = stats {
        let elapsed = Instant::now().duration_since(started_at);
        // We don't care if we couldn't print this successfully.
        let _ = searcher.print_stats(elapsed, stats);
    }
    Ok(matched)
}

```

于是我想在遍历路径的地方，加一个log， 想知道为什么他就没有搜索```common```路径。
在各种百度和google的情况下。还是不知到rust语言是如何加log。但是我看到了源码里有一个```logger```模块.
在代码里发现这样的code
```rust
fn enabled(&self, _: &log::Metadata) -> bool {
    // We set the log level via log::set_max_level, so we don't need to
    // implement filtering here.
    true
}
```
于是我全局搜了一下```set_max_level```这个

```rust
/// Parse the command line arguments for this process.
///
/// If a CLI usage error occurred, then exit the process and print a usage
/// or error message. Similarly, if the user requested the version of
/// ripgrep, then print the version and exit.
///
/// Also, initialize a global logger.
pub fn parse() -> Result<Args> {
    // We parse the args given on CLI. This does not include args from
    // the config. We use the CLI args as an initial configuration while
    // trying to parse config files. If a config file exists and has
    // arguments, then we re-parse argv, otherwise we just use the matches
    // we have here.
    let early_matches = ArgMatches::new(app::app().get_matches());
    set_messages(!early_matches.is_present("no-messages"));
    set_ignore_messages(!early_matches.is_present("no-ignore-messages"));
	
    if let Err(err) = Logger::init() {
        return Err(format!("failed to initialize logger: {}", err).into());
    }
    if early_matches.is_present("trace") {
        log::set_max_level(log::LevelFilter::Trace);
    } else if early_matches.is_present("debug") {
        log::set_max_level(log::LevelFilter::Debug);
    } else {
        log::set_max_level(log::LevelFilter::Warn);
    }

    let matches = early_matches.reconfigure();
    // The logging level may have changed if we brought in additional
    // arguments from a configuration file, so recheck it and set the log
    // level as appropriate.
    if matches.is_present("trace") {
        log::set_max_level(log::LevelFilter::Trace);
    } else if matches.is_present("debug") {
        log::set_max_level(log::LevelFilter::Debug);
    } else {
        log::set_max_level(log::LevelFilter::Warn);
    }
    set_messages(!matches.is_present("no-messages"));
    set_ignore_messages(!matches.is_present("no-ignore-messages"));
    matches.to_args()
}

```
从代码里来看```rg```命令似乎有debug和trace参数。
于是我```help```了一下这个命令
```
--debug                             
		Show debug messages. Please use this when filing a bug report.
		
		The --debug flag is generally useful for figuring out why ripgrep skipped
		searching a particular file. The debug messages should mention all files
		skipped and why they were skipped.
		
		To get even more debug output, use the --trace flag, which implies --debug
		along with additional trace data. With --trace, the output could be quite
		large and is generally more useful for development.

```
于是接下来使用了这个命令
使用rg命令默认的输出log的命令搜索改
```
rg config_screenBrightnessDoze -c --trace 2>&1 | tee result.log
```

### 查看log
搜索出的结果是这样的
```
DEBUG|grep_regex::literal|grep-regex/src/literal.rs:58: literal prefixes detected: Literals { lits: [Complete(config_screenBrightnessDoze)], limit_size: 250, limit_class: 10 }
DEBUG|globset|globset/src/lib.rs:429: built glob set; 0 literals, 0 basenames, 8 extensions, 0 prefixes, 0 suffixes, 0 required extensions, 0 regexes
DEBUG|globset|globset/src/lib.rs:429: built glob set; 0 literals, 0 basenames, 8 extensions, 0 prefixes, 0 suffixes, 0 required extensions, 0 regexes
DEBUG|globset|globset/src/lib.rs:429: built glob set; 0 literals, 0 basenames, 8 extensions, 0 prefixes, 0 suffixes, 0 required extensions, 0 regexes
DEBUG|globset|globset/src/lib.rs:429: built glob set; 0 literals, 3 basenames, 0 extensions, 0 prefixes, 0 suffixes, 0 required extensions, 0 regexes
DEBUG|globset|globset/src/lib.rs:429: built glob set; 0 literals, 0 basenames, 8 extensions, 0 prefixes, 0 suffixes, 0 required extensions, 0 regexes
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("vold.fstab"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./.git: Ignore(IgnoreMatch(Hidden))
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./opensource: Ignore(IgnoreMatch(Gitignore(Glob { from: Some("./.gitignore"), original: "opensource", actual: "**/opensource", is_whitelist: false, is_only_dir: false })))
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./.gitignore: Ignore(IgnoreMatch(Hidden))
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./common: Ignore(IgnoreMatch(Gitignore(Glob { from: Some("./.gitignore"), original: "common", actual: "**/common", is_whitelist: false, is_only_dir: false })))
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./sepolicy: Ignore(IgnoreMatch(Gitignore(Glob { from: Some("./.gitignore"), original: "sepolicy", actual: "**/sepolicy", is_whitelist: false, is_only_dir: false })))
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("AndroidBoard.mk"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("pixart_pat9125.idc"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("hostapd.conf"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("mixer_paths_skuc.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
DEBUG|globset|globset/src/lib.rs:429: built glob set; 0 literals, 1 basenames, 0 extensions, 0 prefixes, 0 suffixes, 0 required extensions, 0 regexes
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./cdrom_res/.gitignore: Ignore(IgnoreMatch(Hidden))
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("ft5x06_ts.kl"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("sound_trigger_platform_info.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("init.qti.synaptics_dsx_qhd.sh"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("mixer_paths_qrd_skuh.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("init.target.rc"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("msm8909w.mk"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("android_filesystem_config.h"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("mixer_paths_qrd_skut.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("mixer_paths_skue.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("system.prop"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("egl.cfg"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("synaptics_dsx.kl"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("recovery.fstab"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("mixer_paths_skua.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("WCNSS_qcom_cfg.ini"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("wpa_supplicant_overlay.conf"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("mixer_paths_qrd_skui.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("Android.mk"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("recovery_nand.fstab"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("gpio-keys.kl"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("mixer_paths.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("BoardConfig.mk"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("init.qcom.modem_links.sh"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("wpa_supplicant.conf"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("result.log"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("listen_platform_info.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("mixer_paths_msm8909_pm8916.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
DEBUG|globset|globset/src/lib.rs:429: built glob set; 0 literals, 0 basenames, 8 extensions, 0 prefixes, 0 suffixes, 0 required extensions, 0 regexes
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("WCNSS_qcom_wlan_nv.bin"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("audio_effects.conf"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("AndroidProducts.mk"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("synaptics_rmi4_i2c.kl"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("hostapd.deny"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("sound_trigger_mixer_paths.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("wearable_core_hardware.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("hostapd.accept"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("fstab.qcom"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("init.carrier.rc"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("atmel_mxt_ts.kl"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("audio_policy.conf"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("WCNSS_wlan_dictionary.dat"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("p2p_supplicant_overlay.conf"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("radio/filesmap"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("cdrom_res/README.txt"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("media/media_codecs_8909.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("media/media_profiles_8909.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("seccomp/mediaextractor-seccomp.policy"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("snd_soc_msm/snd_soc_msm_Taiko_Fluid"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("seccomp/mediacodec-seccomp.policy"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("snd_soc_msm/snd_soc_msm_Taiko_CDP"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("snd_soc_msm/snd_soc_msm_Taiko_liquid"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("snd_soc_msm/snd_soc_msm_Taiko"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("sensors/hals.conf"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/packages/apps/Settings/res/values/styles.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/packages/apps/Settings/res/values/bools.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/packages/apps/Dialer/InCallUI/res/values/styles.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/frameworks/base/core/res/res/values-watch/themes_material.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/packages/apps/Dialer/InCallUI/res/values/colors.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/frameworks/base/core/res/res/xml/power_profile.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("product/overlay/frameworks/base/core/res/res/values/config.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: product/overlay/frameworks/base/core/res/res/values/config.xml:1
Some("overlay/frameworks/base/core/res/res/layout-watch/alert_dialog_material.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/frameworks/base/core/res/res/values/config.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/frameworks/base/core/res/res/values/themes.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/frameworks/base/core/res/res/values/themes_material.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/frameworks/base/core/res/res/values/styles.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:639: Some("overlay/frameworks/base/packages/SystemUI/res/values/config.xml"): searching using generic reader
TRACE|grep_searcher::searcher|grep-searcher/src/searcher/mod.rs:685: generic reader: searching via roll buffer strategy
TRACE|grep_searcher::searcher::core|grep-searcher/src/searcher/core.rs:61: searcher core: will use fast line searcher
result.log:1
```


### 分析log
上面是搜索的trace文件，在trace文件里我们可以看到。有一个文件
```
product/overlay/frameworks/base/core/res/res/values/config.xml
```
是执行了搜索的操作的。
从再看一下这个搜索的trace文件开始的地方
```
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./.git: Ignore(IgnoreMatch(Hidden))
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./opensource: Ignore(IgnoreMatch(Gitignore(Glob { from: Some("./.gitignore"), original: "opensource", actual: "**/opensource", is_whitelist: false, is_only_dir: false })))
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./.gitignore: Ignore(IgnoreMatch(Hidden))
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./common: Ignore(IgnoreMatch(Gitignore(Glob { from: Some("./.gitignore"), original: "common", actual: "**/common", is_whitelist: false, is_only_dir: false })))
DEBUG|ignore::walk|ignore/src/walk.rs:1611: ignoring ./sepolicy: Ignore(IgnoreMatch(Gitignore(Glob { from: Some("./.gitignore"), original: "sepolicy", actual: "**/sepolicy", is_whitelist: false, is_only_dir: false })))
```
很明显在搜索他忽略了```common```, ```sepolicy```, ```opensource```三个目录，
从log的意思来看，他是从```.gitignore```里忽略了这三个目录。
我打开这个文件的的```.gitignore```文件，发现了他的确忽略了这三个目录。

### 验证之前的想法
```
ubuntu14@ubuntu14:msm8909w$ cat .gitignore 
sepolicy
common
opensource
``` 
于是我把这个文件的文件内容删除掉. 
再次搜索，
```
ubuntu14@ubuntu14:msm8909w$ rg config_screenBrightnessDoze 
product/overlay/frameworks/base/core/res/res/values/config.xml
38:    <integer name="config_screenBrightnessDoze">17</integer>
common/product/overlay/frameworks/base/core/res/res/values/config.xml
45:    <integer name="config_screenBrightnessDoze">17</integer>
```
果然，出现了搜索出了```common```目录下的文件。
如果使用```--no-ignore ```参数搜索也可以有。
```
ubuntu14@ubuntu14:msm8909w$ rg config_screenBrightnessDoze  --no-ignore
product/overlay/frameworks/base/core/res/res/values/config.xml
38:    <integer name="config_screenBrightnessDoze">17</integer>

common/product/overlay/frameworks/base/core/res/res/values/config.xml
45:    <integer name="config_screenBrightnessDoze">17</integer>
```


## 总结
原来 rg 搜索， 默认会忽略掉```.gitignore```文件里的忽略的文件。 也就是说，对一个git仓库，ripgrep 会默认不搜索不在版本控制的文件。
