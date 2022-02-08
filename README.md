# OpenCL-Wrapper
OpenCL is the most powerful programming language ever created. Yet the OpenCL C++ bindings are very cumbersome and the code overhead prevents many people from getting started.
I created this lightweight OpenCL-Wrapper to greatly simplify OpenCL software development with C++ while keeping functionality and performance.

Works in both Windows and Linux with C++17.

## Key simplifications:
1. select a `Device` with 1 line
   - automatically select fastest device / device with most memory / device with specified ID from a list of all devices
   - easily get device information (performance in TFLOPs/s, amount of memory and cache, FP64/FP16 capabilities, etc.)
   - automatic OpenCL C code compilation when creating the Device object
     - automatically enable FP64/FP16 capabilities in OpenCL C code
     - automatically print log to console if there are compile errors
     - easy option to generate PTX assembly and save that in a `.ptx` file
2. create a `Memory` object with 1 line
   - one object for both host and device memory
   - easy host <-> device memory transfer
   - easy handling of multi-dimensional vectors
   - can also be used to only allocate memory on host or only allocate memory on device
3. create a `Kernel` with 1 line
   - Memory objects are linked to OpenCL C kernel parameters during Kernel creation
   - easy Kernel execution
4. OpenCL C code is embedded into C++
   - syntax highlighting in the code editor is retained

## No need to:
- have code overhead for selecting a platform/device, passing the OpenCL C code, etc.
- keep track of global/local ranges for buffers and kernels
- have duplicate code for host and device buffers
- bother with Queue, Context, Source, Program
- load a `.cl` file at runtime

## Example (OpenCL vector addition)
### main.cpp
```c
#include "opencl.hpp"

int main() {
	const Device device(select_device_with_most_flops()); // compile OpenCL C code for the fastest available device

	const uint N = 1024u; // size of vectors

	Memory<float> A(device, N); // allocate memory on both host and device
	Memory<float> B(device, N);
	Memory<float> C(device, N);

	for(uint n=0u; n<N; n++) {
		A[n] = 3.0f; // initialize memory
		B[n] = 2.0f;
		C[n] = 1.0f;
	}

	const Kernel add_kernel(device, N, "add_kernel", A, B, C); // kernel that runs on the device

	print_info("Value before kernel execution: C[0] = "+to_string(C[0]));

	A.write_to_device(); // copy data from host memory to device memory
	B.write_to_device(); // copy data from host memory to device memory
	add_kernel.run(); // run add_kernel on the device
	C.read_from_device(); // copy data from device memory to host memory

	print_info("Value after kernel execution: C[0] = "+to_string(C[0]));

	wait();
	return 0;
}
```

### kernel.cpp
```c
#include "kernel.hpp" // note: string literals can't be arbitrarily long, so periodically interrupt with )+R(
string opencl_c_container() { return R( // ########################## begin of OpenCL C code ####################################################################



kernel void add_kernel(global float* A, global float* B, global float* C) { // equivalent to "for(int n=0; n<N; n++) {", but executed in parallel
	const uint n = get_global_id(0);
	C[n] = A[n]+B[n];
}



);} // ############################################################### end of OpenCL C code #####################################################################
```

### For comparison, the very same OpenCL vector addition example, containing 35 lines of program logic, looks like this when directly using the OpenCL C++ bindings:
```c
#define to_string to_string_old // use own to_string methods
#include <CL/cl.hpp>
#undef to_string // use own to_string methods
#include "utilities.hpp"

#define THREAD_BLOCK_SIZE 32

int main() {

	// 1. select device

	vector<cl::Device> cl_devices; // get all devices of all platforms
	{
		vector<cl::Platform> cl_platforms; // get all platforms (drivers)
		cl::Platform::get(&cl_platforms);
		for(uint i=0u; i<(uint)cl_platforms.size(); i++) {
			vector<cl::Device> cl_devices_available;
			cl_platforms[i].getDevices(CL_DEVICE_TYPE_ALL, &cl_devices_available);
			for(uint j=0u; j<(uint)cl_devices_available.size(); j++) {
				cl_devices.push_back(cl_devices_available[j]);
			}
		}
	}
	cl::Device cl_device; // select fastest available device
	{
		float best_value = 0.0f;
		uint best_i = 0u; // index of fastest device
		for(uint i=0u; i<(uint)cl_devices.size(); i++) { // find device with highest (estimated) floating point performance
			const string vendor = trim(cl_devices[i].getInfo<CL_DEVICE_VENDOR>()); // device vendor
			const uint compute_units = (uint)cl_devices[i].getInfo<CL_DEVICE_MAX_COMPUTE_UNITS>(); // compute units (CUs) can contain multiple cores depending on the microarchitecture
			const uint clock_frequency = (uint)cl_devices[i].getInfo<CL_DEVICE_MAX_CLOCK_FREQUENCY>(); // in MHz
			bool is_cpu = cl_devices[i].getInfo<CL_DEVICE_TYPE>()==CL_DEVICE_TYPE_CPU;
			bool is_gpu = cl_devices[i].getInfo<CL_DEVICE_TYPE>()==CL_DEVICE_TYPE_GPU;
			const uint ipc = is_gpu?2u:32u; // IPC (instructions per cycle) is 2 for GPUs and 32 for most modern CPUs
			const float nvidia = (float)(contains(to_lower(vendor), "nvidia"))*(clock_frequency<1000u?192.0f:compute_units<=30u?128.0f:64.0f); // Nvidia GPUs have 192 cores/CU (Kepler), 128 cores/CU (Maxwell, Pascal, Ampere) or 64 cores/CU (P100, Volta, Turing, A100)
			const float amd = (float)(contains(to_lower(vendor), "amd")||contains(to_lower(vendor), "advanced"))*(is_gpu?64.0f:0.5f); // AMD GCN GPUs usually have 64 cores/CU, AMD CPUs have 1/2 core/CU
			const float intel = (float)(contains(to_lower(vendor), "intel"))*(is_gpu?8.0f:0.5f); // Intel integrated GPUs usually have 8 cores/CU, Intel CPUs have 1/2 core/CU
			const float arm = (float)(contains(to_lower(vendor), "arm"))*(is_gpu?8.0f:1.0f); // ARM GPUs usually have 8 cores/CU, ARM CPUs have 1 core/CU
			const uint cores = to_uint((float)compute_units*(nvidia+amd+intel+arm)); // for CPUs, compute_units is the number of threads (twice the number of cores with hyperthreading)
			const float tflops = 1E-6f*(float)cores*(float)ipc*(float)clock_frequency; // estimated device floating point performance in TeraFLOPs/s
			if(tflops>best_value) { // device_memory>best_value
				best_value = tflops; // best_value = device_memory;
				best_i = i; // find index of fastest device
			}
		}
		const string name = trim(cl_devices[best_i].getInfo<CL_DEVICE_NAME>()); // device name
		cl_device = cl_devices[best_i];
		print_info(name); // print device name
	}

	// 2. embed OpenCL C code (raw string literal breaks syntax highlighting)

	string opencl_c_code = R"(
		kernel void add_kernel(global float* A, global float* B, global float* C) { // equivalent to "for(int n=0; n<N; n++) {", but executed in parallel
			const uint n = get_global_id(0);
			C[n] = A[n]+B[n];
		}
	)";

	// 3. compile OpenCL C code

	cl::Context cl_context;
	cl::Program cl_program;
	cl::CommandQueue cl_queue;
	{
		cl_context = cl::Context(cl_device);
		cl_queue = cl::CommandQueue(cl_context, cl_device);
		cl::CommandQueue cl_queue(cl_context, cl_device); // queue to push commands for the device
		cl::Program::Sources cl_source;
		cl_source.push_back({ opencl_c_code.c_str(), opencl_c_code.length() });
		cl_program = cl::Program(cl_context, cl_source);
		int error = cl_program.build("-cl-fast-relaxed-math -w"); // compile OpenCL C code, disable warnings
		if(error) print_warning(cl_program.getBuildInfo<CL_PROGRAM_BUILD_LOG>(cl_device)); // print build log
		if(error) print_error("OpenCL C code compilation failed. Make sure there are no errors in kernel.cpp (\"#define LOG\" might help). If your GPU is old, try uncommenting \"#define USE_OPENCL_1_1\".");
		else print_info("OpenCL C code successfully compiled.");
	}

	// 4. allocate memory on host and device

	const uint N = 1024u;
	float* host_A;
	float* host_B;
	float* host_C;
	cl::Buffer device_A;
	cl::Buffer device_B;
	cl::Buffer device_C;
	{
		host_A = new float[N];
		host_B = new float[N];
		host_C = new float[N];
		for(uint i=0u; i<N; i++) {
			host_A[i] = 0.0f; // zero all buffers
			host_B[i] = 0.0f;
			host_C[i] = 0.0f;
		}
		int error = 0;
		device_A = cl::Buffer(cl_context, CL_MEM_READ_WRITE, N*sizeof(float), nullptr, &error);
		if(error) print_error("OpenCL Buffer allocation failed with error code "+to_string(error)+".");
		device_B = cl::Buffer(cl_context, CL_MEM_READ_WRITE, N*sizeof(float), nullptr, &error);
		if(error) print_error("OpenCL Buffer allocation failed with error code "+to_string(error)+".");
		device_C = cl::Buffer(cl_context, CL_MEM_READ_WRITE, N*sizeof(float), nullptr, &error);
		if(error) print_error("OpenCL Buffer allocation failed with error code "+to_string(error)+".");
		cl_queue.enqueueWriteBuffer(device_A, true, 0u, N*sizeof(float), (void*)host_A); // have to keep track of buffer range and buffer data type
		cl_queue.enqueueWriteBuffer(device_B, true, 0u, N*sizeof(float), (void*)host_B);
		cl_queue.enqueueWriteBuffer(device_C, true, 0u, N*sizeof(float), (void*)host_C);
	}

	// 5. create Kernel object and link input parameters

	cl::NDRange cl_range_global, cl_range_local;
	cl::Kernel cl_kernel;
	{
		cl_kernel = cl::Kernel(cl_program, "add_kernel");
		cl_kernel.setArg(0, device_A);
		cl_kernel.setArg(1, device_B);
		cl_kernel.setArg(2, device_C);
		cl_range_local = cl::NDRange(THREAD_BLOCK_SIZE); // warp size is 32
		cl_range_global = cl::NDRange(((N+THREAD_BLOCK_SIZE-1)/THREAD_BLOCK_SIZE)*THREAD_BLOCK_SIZE); // make global range a multiple of local range
	}

	// 6. finally run the actual program

	{
		for(uint i=0u; i<N; i++) {
			host_A[i] = 3.0f; // initialize buffers on host
			host_B[i] = 2.0f;
			host_C[i] = 1.0f;
		}

		print_info("Value before kernel execution: C[0] = "+to_string(host_C[0]));

		cl_queue.enqueueWriteBuffer(device_A, true, 0u, N*sizeof(float), (void*)host_A); // copy A and B to device
		cl_queue.enqueueWriteBuffer(device_B, true, 0u, N*sizeof(float), (void*)host_B); // have to keep track of buffer range and buffer data type
		cl_queue.enqueueNDRangeKernel(cl_kernel, cl::NullRange, cl_range_global, cl_range_local); // have to keep track of kernel ranges
		cl_queue.finish(); // don't forget to finish the queue
		cl_queue.enqueueReadBuffer(device_C, true, 0u, N*sizeof(float), (void*)host_C);

		print_info("Value after kernel execution: C[0] = "+to_string(host_C[0]));
	}

	wait();
	return 0;
}
```