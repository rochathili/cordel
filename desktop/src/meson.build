cordel_sources = [
  'main.vala',
  'application.vala',
  'window.vala',
]

cordel_deps = [
  config_dep,
  dependency('gtk4'),
  dependency('libadwaita-1', version: '>= 1.4'),
]

cordel_sources += gnome.compile_resources('cordel-resources',
  'cordel.gresource.xml',
  c_name: 'cordel'
)

executable('cordel', cordel_sources,
  dependencies: cordel_deps,
  include_directories: config_inc,
       install: true,
)
