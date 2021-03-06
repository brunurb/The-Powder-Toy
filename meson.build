project('the-powder-toy', [ 'c', 'cpp' ], version: 'the.cake.is.a.lie', default_options: [
	'cpp_std=c++11',
])

to_array = generator(
	import('python').find_installation('python3'),
	output: [ '@PLAINNAME@.cpp', '@PLAINNAME@.h' ],
	arguments: [ join_paths(meson.current_source_dir(), 'to_array.py'), '@BUILD_DIR@', '@OUTPUT0@', '@OUTPUT1@', '@INPUT@', '@EXTRA_ARGS@' ]
)

cpp_compiler = meson.get_compiler('cpp')

conf_data = configuration_data()
conf_data.set('CURL_STATICLIB', false)
conf_data.set('ZLIB_WINAPI', false)

copt_x86 = host_machine.cpu_family() in [ 'x86_64', 'x86' ]
copt_64bit = host_machine.cpu_family() in [ 'x86_64', 'aarch64' ]
copt_msvc = cpp_compiler.get_id() in [ 'msvc' ]

if host_machine.system() in [ 'linux', 'freebsd' ]
	copt_platform = 'lin'
elif host_machine.system() in [ 'windows' ]
	copt_platform = 'win'
elif host_machine.system() in [ 'darwin' ]
	copt_platform = 'mac'
else
	error('unsupported platform: ' + host_machine.system())
endif

if get_option('ogli') or get_option('oglr')
	error('OpenGL features are currently unavailable')
endif

uopt_static = get_option('static')
use_tpt_libs = 'no'
tpt_libs_vtag = 'v20210103095432'
if uopt_static == 'system'
	if copt_platform == 'win'
		error('no way to find static system libraries on windows')
	endif
elif uopt_static == 'prebuilt'
	use_tpt_libs = 'static'
else
	if copt_platform == 'win'
		use_tpt_libs = 'dynamic'
	endif
endif
if use_tpt_libs != 'no'
	if host_machine.cpu_family() != 'x86_64'
		error('we do not currently provide prebuilt @0@ libraries for @1@ on @2@'.format(use_tpt_libs, host_machine.system(), host_machine.cpu_family()))
	endif
	tpt_libs = subproject('tpt-libs-prebuilt-@0@64-@1@-@2@'.format(copt_platform, use_tpt_libs, tpt_libs_vtag))
endif

uopt_native = get_option('native')
uopt_x86_sse = get_option('x86_sse')
if uopt_x86_sse == 'auto'
	uopt_x86_sse_level = 20
elif uopt_x86_sse == 'sse3'
	uopt_x86_sse_level = 30
elif uopt_x86_sse == 'sse2'
	uopt_x86_sse_level = 20
elif uopt_x86_sse == 'sse'
	uopt_x86_sse_level = 10
elif uopt_x86_sse == 'none'
	uopt_x86_sse_level = 0
endif
if not copt_x86 or uopt_native
	uopt_x86_sse_level = 0
endif

uopt_lua = get_option('lua')
if uopt_lua == 'luajit'
	lua_opt_dep = [ use_tpt_libs != 'no' ? tpt_libs.get_variable('luajit_dep') : dependency('luajit', static: uopt_static == 'system') ]
elif uopt_lua == 'lua5.2'
	lua_opt_dep = [ use_tpt_libs != 'no' ? tpt_libs.get_variable('lua52_dep') : dependency('lua5.2-c++', static: uopt_static == 'system') ]
elif uopt_lua == 'lua5.1'
	lua_opt_dep = [ use_tpt_libs != 'no' ? tpt_libs.get_variable('lua51_dep') : dependency('lua5.1-c++', static: uopt_static == 'system') ]
else
	lua_opt_dep = []
endif

uopt_http = get_option('http')
if uopt_http
	curl_opt_dep = [ use_tpt_libs != 'no' ? tpt_libs.get_variable('libcurl_dep') : dependency('libcurl', static: uopt_static == 'system') ]
else
	curl_opt_dep = []
endif

uopt_fftw = get_option('gravfft')
if uopt_fftw
	fftw_opt_dep = [ use_tpt_libs != 'no' ? tpt_libs.get_variable('fftw_dep') : dependency('fftw3f', static: uopt_static == 'system') ]
else
	fftw_opt_dep = []
endif

threads_dep = dependency('threads')
zlib_dep = use_tpt_libs != 'no' ? tpt_libs.get_variable('zlib_dep') : dependency('zlib', static: uopt_static == 'system')
sdl2_dep = use_tpt_libs != 'no' ? tpt_libs.get_variable('sdl2_dep') : dependency('sdl2', static: uopt_static == 'system')

project_link_args = []
project_c_args = []
project_cpp_args = []
if copt_msvc
	if uopt_x86_sse_level >= 30
		message('SSE3 configured to be enabled but unavailable in msvc')
		uopt_x86_sse_level = 20
	endif
	if uopt_native
		message('local machine optimization configured to be enabled but unavailable in msvc')
		uopt_native = false
	endif
	args_msvc = [
		'/GS',
		'-D_SCL_SECURE_NO_WARNINGS',
	]
	if copt_64bit
		message('SSE explicitly configured but unavailable in msvc targeting 64-bit machines')
	else
		if uopt_x86_sse_level >= 20
			args_msvc += [ '/arch:SSE2' ]
		elif uopt_x86_sse_level >= 10
			args_msvc += [ '/arch:SSE' ]
		endif
	endif
	if get_option('debug')
		project_link_args += [
			'/NODEFAULTLIB:MSVCRT',
		]
	else
		args_msvc += [
			'/Oy-',
			'/fp:fast',
		]
		project_link_args += [
			'/OPT:REF',
			'/OPT:ICF',
		]
	endif
	project_c_args += args_msvc
	project_cpp_args += args_msvc
else
	args_ccomp = [ '-U__STRICT_ANSI__', '-Wno-unused-result' ]
	if copt_platform == 'mac'
		if uopt_x86_sse_level >= 0
			message('SSE level explicitly configured but unavailable on macosx')
			uopt_x86_sse_level = 0
		endif
		if uopt_native
			message('local machine optimization configured to be enabled but unavailable on macosx')
			uopt_native = false
		endif
	else
		if uopt_x86_sse_level >= 30
			args_ccomp += [ '-msse3' ]
		endif
		if uopt_x86_sse_level >= 20
			args_ccomp += [ '-msse2' ]
		endif
		if uopt_x86_sse_level >= 10
			args_ccomp += [ '-msse' ]
		endif
		if uopt_native
			args_ccomp += [ '-march=native' ]
		endif
	endif
	if not get_option('debug')
		args_ccomp += [
			'-ftree-vectorize',
			'-funsafe-math-optimizations',
			'-ffast-math',
			'-fomit-frame-pointer',
		]
	endif
	project_c_args += args_ccomp
	project_cpp_args += args_ccomp + [ '-Wno-invalid-offsetof' ]
endif

if copt_platform == 'win'
	other_dep = tpt_libs.get_variable('other_dep')
	sdl2main_dep = tpt_libs.get_variable('sdl2main_dep')
	args_ccomp_win = [ '-D_WIN32_WINNT=0x0501' ]
	project_c_args   += args_ccomp_win
	project_cpp_args += args_ccomp_win
	windows_mod = import('windows')
	if uopt_static != 'none'
		conf_data.set('CURL_STATICLIB', true)
		conf_data.set('ZLIB_WINAPI', true)
	else
		foreach input_and_output : tpt_libs.get_variable('config_dlls')
			configure_file(input: input_and_output[0], output: input_and_output[1], copy: true)
		endforeach
	endif
endif

if copt_platform == 'mac' and uopt_lua == 'luajit'
	project_link_args += [
		'-pagezero_size', '10000',
		'-image_base', '100000000',
	]
endif

project_inc = include_directories([ 'src', 'data', 'resources' ])

conf_data.set('LIN', copt_platform == 'lin')
conf_data.set('WIN', copt_platform == 'win')
conf_data.set('MACOSX', copt_platform == 'mac')
conf_data.set('X86', copt_x86)
conf_data.set('X86_SSE3', uopt_x86_sse_level >= 30)
conf_data.set('X86_SSE2', uopt_x86_sse_level >= 20)
conf_data.set('X86_SSE', uopt_x86_sse_level >= 10)
conf_data.set('NATIVE', uopt_native)
conf_data.set('_64BIT', copt_64bit)
conf_data.set('OGLI', get_option('ogli'))
conf_data.set('OGLR', get_option('oglr'))
conf_data.set('PIX32OGL', get_option('ogli'))
conf_data.set('BETA', get_option('beta'))
conf_data.set('NO_INSTALL_CHECK', not get_option('install_check'))
conf_data.set('IGNORE_UPDATES', get_option('ignore_updates'))
conf_data.set('SAVE_VERSION', get_option('version_major'))
conf_data.set('MINOR_VERSION', get_option('version_minor'))
conf_data.set('BUILD_NUM', get_option('version_build'))
conf_data.set('MOD_ID', get_option('mod_id'))
conf_data.set('DEBUG', get_option('debug'))
conf_data.set('SNAPSHOT', get_option('snapshot'))
conf_data.set('SNAPSHOT_ID', get_option('snapshot_id'))
conf_data.set('FUTURE_SAVE_VERSION', get_option('future_major'))
conf_data.set('FUTURE_MINOR_VERSION', get_option('future_minor'))
conf_data.set('SERVER', '"' + get_option('server') + '"')
conf_data.set('STATICSERVER', '"' + get_option('static_server') + '"')
if get_option('update_server') != ''
	conf_data.set('UPDATESERVER', '"' + get_option('update_server') + '"')
else
	conf_data.set('UPDATESERVER', false)
endif

resources_files = []

subdir('src')
subdir('data')
subdir('resources')

if get_option('build_powder')
	powder_deps = [
		threads_dep,
		zlib_dep,
		sdl2_dep,
		lua_opt_dep,
		curl_opt_dep,
		fftw_opt_dep,
	]
	if copt_platform == 'win'
		powder_deps += other_dep
		powder_deps += sdl2main_dep
	endif
	executable(
		'powder',
		sources: powder_files,
		include_directories: [ project_inc, powder_inc ],
		c_args: project_c_args,
		cpp_args: project_cpp_args,
		cpp_pch: 'pch/pch_cpp.h',
		gui_app: true,
		link_args: project_link_args,
		dependencies: powder_deps,
	)
endif

if get_option('build_render')
	render_deps = [
		threads_dep,
		zlib_dep,
	]
	executable(
		'render',
		sources: render_files,
		include_directories: [ project_inc, render_inc ],
		c_args: project_c_args,
		cpp_args: project_cpp_args,
		cpp_pch: 'pch/pch_cpp.h',
		link_args: project_link_args,
		dependencies: render_deps,
	)
endif

if get_option('build_font')
	font_deps = [
		threads_dep,
		zlib_dep,
		sdl2_dep,
	]
	if copt_platform == 'win'
		font_deps += other_dep
		font_deps += sdl2main_dep
	endif
	executable(
		'font',
		sources: font_files,
		include_directories: [ project_inc, font_inc ],
		c_args: project_c_args,
		cpp_args: project_cpp_args,
		cpp_pch: 'pch/pch_cpp.h',
		gui_app: true,
		link_args: project_link_args,
		dependencies: font_deps,
	)
endif
