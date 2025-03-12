# wasm_sample
A sample wasm lib


# Sample for calling wasm from KMP desktop app
```kotlin
import java.io.File
import java.io.IOException
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.util.concurrent.TimeUnit


/**
 * A class for loading and executing WebAssembly files in a Kotlin Multiplatform desktop application
 * by using external WebAssembly runtimes (wasmtime or wasmer) via ProcessBuilder.
 */
class WasmLoader {
    // Path to the WebAssembly runtime executable
    private val wasmRuntimePath: String = findWasmRuntime()

    /**
     * Finds a WebAssembly runtime (wasmtime or wasmer) in the system PATH
     * @return Path to the WebAssembly runtime executable
     * @throws IllegalStateException if no WebAssembly runtime is found
     */
    private fun findWasmRuntime(): String {
        // Check for wasmtime
        val wasmtimeProcess = ProcessBuilder("which", "wasmtime")
            .redirectOutput(ProcessBuilder.Redirect.PIPE)
            .start()

        var path = wasmtimeProcess.inputStream.bufferedReader().readLine()
        wasmtimeProcess.waitFor(1, TimeUnit.SECONDS)

        if (path?.isNotEmpty() == true) {
            println("Found wasmtime at: $path")
            return path
        }

        // Check for wasmer
        val wasmerProcess = ProcessBuilder("which", "wasmer")
            .redirectOutput(ProcessBuilder.Redirect.PIPE)
            .start()

        path = wasmerProcess.inputStream.bufferedReader().readLine()
        wasmerProcess.waitFor(1, TimeUnit.SECONDS)

        if (path?.isNotEmpty() == true) {
            println("Found wasmer at: $path")
            return path
        }

        // Windows-specific check
        if (System.getProperty("os.name").lowercase().contains("win")) {
            // Check common installation locations on Windows
            val possibleLocations = listOf(
                "C:\\Program Files\\wasmtime\\wasmtime.exe",
                "C:\\Program Files (x86)\\wasmtime\\wasmtime.exe",
                "C:\\Program Files\\wasmer\\wasmer.exe",
                "C:\\Program Files (x86)\\wasmer\\wasmer.exe"
            )

            for (location in possibleLocations) {
                if (File(location).exists()) {
                    println("Found WebAssembly runtime at: $location")
                    return location
                }
            }
        }

        throw IllegalStateException(
            "No WebAssembly runtime found. Please install wasmtime or wasmer and make sure it's in your PATH."
        )
    }

    /**
     * Executes a WebAssembly file with the specified function and arguments
     * @param wasmFilePath Path to the WebAssembly file
     * @param functionName Name of the function to call
     * @param args Arguments to pass to the function
     * @return The output of the WebAssembly function execution
     */
    /**
     * Executes a WebAssembly function with validated numeric inputs and returns the result as a String
     * @param wasmFilePath Absolute path to the .wasm file
     * @param functionName Exported function name to call
     * @param args Numeric arguments for the function
     * @return Result of the calculation as String
     * @throws IllegalArgumentException for invalid inputs
     * @throws RuntimeException for execution errors
     */
    suspend fun executeWasmFunction(
        wasmFilePath: String,
        functionName: String,
        vararg args: String
    ): String = withContext(Dispatchers.IO) {
        // Validate WASM file existence
        val wasmFile = File(wasmFilePath)
        if (!wasmFile.exists() || !wasmFile.canRead()) {
            throw IllegalArgumentException(
                "WebAssembly file not found or not readable: ${wasmFile.absolutePath}"
            )
        }

        // Validate numeric arguments
        args.forEach { arg ->
            if (!arg.matches(Regex("^-?\\d+$"))) {
                throw IllegalArgumentException(
                    "Invalid argument '$arg' - only integer numbers allowed"
                )
            }
        }

        // Construct the command
        val command = mutableListOf(
            wasmRuntimePath,
            "run",
            "--invoke",
            functionName,
            wasmFile.absolutePath
        ).apply {
            addAll(args)
        }

        println("cmd: $command")

        // Execute the process with timeout
        val process = try {
            ProcessBuilder(command)
                .redirectOutput(ProcessBuilder.Redirect.PIPE)
                .redirectError(ProcessBuilder.Redirect.PIPE)
                .start()
        } catch (e: IOException) {
            throw RuntimeException("Failed to start WebAssembly runtime: ${e.message}")
        }

        // Read output streams
        val output = try {
            process.inputStream.bufferedReader().use { it.readText() }
        } catch (e: IOException) {
            throw RuntimeException("Failed to read WASM output: ${e.message}")
        }

        val error = try {
            process.errorStream.bufferedReader().use { it.readText() }
        } catch (e: IOException) {
            throw RuntimeException("Failed to read WASM error stream: ${e.message}")
        }

        // Wait for completion with timeout
        val exitCode = try {
            if (!process.waitFor(5, TimeUnit.SECONDS)) {
                process.destroy()
                throw RuntimeException("WASM execution timed out after 5 seconds")
            }
            process.exitValue()
        } catch (e: InterruptedException) {
            process.destroy()
            throw RuntimeException("Execution interrupted: ${e.message}")
        }

        // Handle non-zero exit codes
        if (exitCode != 0) {
            val errorMessage = when (exitCode) {
                1 -> "WASM function trap (e.g., division by zero)"
                2 -> "Invalid command line arguments"
                3 -> "Function not found in module"
                else -> "Unknown error (exit code $exitCode)"
            }
            throw RuntimeException(
                "WebAssembly execution failed: $errorMessage\nDetails: $error"
            )
        }

        // Parse and validate the numeric result
        val rawResult = output.trim()
            .replace(Regex("[^\\d-]"), " ") // Remove non-numeric characters
            .split(Regex("\\s+"))
            .firstOrNull { it.isNotEmpty() }
            ?: throw RuntimeException("No valid numeric result found in output: '$output'")

        return@withContext when {
            rawResult.matches(Regex("^-?\\d+$")) -> rawResult
            else -> throw NumberFormatException("Invalid numeric result: '$rawResult'")
        }
    }

    /**
     * Example usage of the WasmLoader
     */
    suspend fun demoWasmUsage() {
        try {
            // Pfad zur kompilierten WASM-Datei
            val wasmPath = "/Users/art/SourceCode/NoCode/wasm_sample/calculate.wasm"

            // Prüfe, ob die Datei existiert
            if (!File(wasmPath).exists()) {
                println("❌ WASM file not found: $wasmPath")
                return
            }

            // Führe die WASM-Funktion aus und erfasse das Ergebnis
            val result = executeWasmFunction(wasmPath, "calculate", "5", "3")

            // Ergebnis ausgeben
            println("✅ Result from WASM function: $result")

        } catch (e: Exception) {
            println("❌ Error executing WASM: ${e.message}")
            e.printStackTrace()
        }
    }
}

suspend fun main() {

    val loader = WasmLoader()
    loader.demoWasmUsage()
}
```