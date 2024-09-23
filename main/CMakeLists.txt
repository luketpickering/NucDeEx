add_custom_target(NucDeExApps)

foreach(app
    plot_decay
    simulation
    cout_input
    )
    add_executable(${app} ${app}.cc)
    add_dependencies(NucDeExApps ${app})
    target_link_libraries(${app} NucDeEx::All)

    install(TARGETS ${app} DESTINATION bin)
endforeach(app)
