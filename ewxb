#!/usr/bin/env bash
# Build a GCC toolchain with Go-support from scratch.
# Author: Erik Westrup <erik.westrup@gmail.com>
# Inspired by Jim Blandy's excellent eglibc cross-compiling guide posted at eglibc's mailinglist as "[patches] Cross-building instructions", available at http://www.eglibc.org/archives/patches/msg00078.html

set -e

scriptname=${0##*/}

# UI
prompt="|>"
phasestars="*********"
funcstars="${phasestars}***"

log () {
	local fmt=""
	if [ "$#"  -eq 1 ]; then
		fmt="%s"
	elif [ "$#"  -gt 1 ]; then
		fmt="$1"
		shift 1
	fi
	printf "%s ${fmt}\n" "|>" "$@"
}

source x_environment.sh


# Log stdout and stderr.
date=$(date "+%Y-%m-%d")
log_file="/tmp/${scriptname}_${date}.log"
exec > >(tee -a "$log_file")
exec 2> >(tee -a "$log_file" >&2)
log "$(date "+%Y-%m-%d-%H:%M:%S") Appending stdout & stdin to: ${log_file}"



glibc_needs_port_pkg() {
	[[ $GLIBCVNO =~ 2\.([3-9]|1[0-6]) ]]
}

setup_and_enter_dir() {
	local dir="$1"
	if [ -d "$dir" ]; then
		printf "%s exists, delete it? [Y/n]: " "$dir"
		read delete
		if ([ -z "$delete" ] || [[ "$delete" = [yY] ]]); then
			rm -rf "$dir"
			mkdir "$dir"
		fi
	else
		mkdir "$dir"
	fi
	cd "$dir"
}

declare -A phases # Assoc array with phase to name mappings.

phases+=(["0"]="prefix")
phase_0() {
	log "Setting up work dirs and fetching/unpacking sources."

	mkdir -p "$CWORK"
	mkdir -p "$SRC"
	rm -rf $OBJ
	rm -rf $TOOLS
	rm -rf $SYSROOT
	mkdir $OBJ
	mkdir $TOOLS
	mkdir $SYSROOT
	mkdir -p $SYSROOT/usr/include

	cd $SRC

	if ! [ -f "$BINUTILSV.tar.gz" ]; then
		log "Downloading $BINUTILSV.tar.gz..."
		wget http://ftp.gnu.org/gnu/binutils/$BINUTILSV.tar.gz
	fi
	if ! [ -d "$BINUTILSV" ]; then
		tar xvzf $BINUTILSV.tar.gz
	fi

	if [ "$GCCV"  == "gccgo" ]; then
		if ! [ -d "$GCCV" ]; then
			log "Checking out $GCCV from SVN repo..."
			svn checkout svn://gcc.gnu.org/svn/gcc/branches/gccgo gccgo
			cd $GCCV
			contrib/download_prerequisites
			cd $SRC
		fi
	else
		if ! [ -f "$GCCV.tar.bz2" ]; then
			log "Downloading $GCCV.tar.bz2..."
			wget http://ftp.gnu.org/gnu/gcc/$GCCV/$GCCV.tar.bz2
		fi
		if ! [ -d "$GCCV" ]; then
			tar xvjf $GCCV.tar.bz2
			cd $GCCV
			contrib/download_prerequisites
			cd $SRC
		fi
	fi

	if ! [ -f "$GLIBCV.tar.bz2" ]; then
		log "Downloading $GLIBCV.tar.gz2..."
		wget http://ftp.gnu.org/gnu/glibc/$GLIBCV.tar.bz2
	fi
	if ! [ -d "$GLIBCV" ]; then
		tar xvjf $GLIBCV.tar.bz2
	fi

	if glibc_needs_port_pkg; then
		cd $GLIBCV
		glibcport="glibc-ports-$GLIBCVNO"
		if ! [ -f "$glibcport.tar.bz2" ]; then
			log "Fetching ports extension $glibcport.tar.bz2"
			wget http://ftp.gnu.org/gnu/glibc/$glibcport.tar.bz2
		fi
		if ! [ -d "$glibcport" ]; then
			tar xvjf $glibcport.tar.bz2
			ln -s $glibcport ports
		fi
		cd ..
	fi

	if ! [ -f "$LINUXV.tar.gz" ]; then
		log "Downloading $LINUXV.tar.gz..."
		wget https://www.kernel.org/pub/linux/kernel/v2.6/$LINUXV.tar.gz
	fi
	if ! [ -d "$LINUXV" ]; then
		tar xvzf $LINUXV.tar.gz
	fi

}

phases+=(["1"]="binutils")
phase_1() {
	log "Building cross-compiling binutils."

	setup_and_enter_dir "$OBJ/binutils"

	$SRC/$BINUTILSV/configure \
		--prefix=$TOOLS \
		--target=$TARGET \
		--with-sysroot=$SYSROOT

	make
	make install
}

phases+=(["2"]="gcc1")
phase_2() {
	log "Building barebone cross GCC so glibc headers can be compiled."

	setup_and_enter_dir "$OBJ/gcc1"

	$SRC/$GCCV/configure \
		--prefix=$TOOLS \
		--build=$BUILD \
		--host=$HOST \
		--target=$TARGET \
		--enable-languages=c \
		--without-headers \
		--with-newlib \
		--with-pkgversion="${USER}'s $TARGET GCC phase1 cross-compiler" \
		--disable-libgcc \
		--disable-shared \
		--disable-threads \
		--disable-libssp \
		--disable-libgomp \
		--disable-libmudflap \
		--disable-libquadmath \
		--disable-libquadmath-support

	PATH="$TOOLS/bin:$PATH" make all-gcc
	PATH="$TOOLS/bin:$PATH" make install-gcc
}

phases+=(["3"]="linux headers")
phase_3() {
	log "Compiling and installing Linux header files."

	rm -rf $OBJ/$LINUXV
	cp -r $SRC/$LINUXV $OBJ # Make modifies the tree; make copy.
	cd $OBJ/$LINUXV

	make clean
	make headers_install \
		ARCH=$LINUX_ARCH \
		CROSS_COMPILE=$TARGET \
		INSTALL_HDR_PATH=$SYSROOT/usr
}

phases+=(["4"]="glibc headers")
phase_4() {
	log "Install header files and bootstrap libc with friends."

	setup_and_enter_dir "$OBJ/glibc-headers"

	LD_LIBRARY_PATH_old="$LD_LIBRARY_PATH"
	unset LD_LIBRARY_PATH

	local addons="--enable-add-ons"
	if glibc_needs_port_pkg; then
		addons="--enable-add-ons=nptl,ports"
	fi

	BUILD_CC=gcc \
	CC=$TOOLS/bin/$TARGET-gcc \
	CXX=$TOOLS/bin/$TARGET-g++ \
	AR=$TOOLS/bin/$TARGET-ar \
	RANLIB=$TOOLS/bin/$TARGET-ranlib \
	$SRC/$GLIBCV/configure \
		--prefix=/usr \
		--build=$BUILD \
		--host=$TARGET \
		--with-headers=$SYSROOT/usr/include \
		--with-binutils=$TOOLS/$TARGET/bin \
		$addons \
		--enable-kernel="${LINUXV##*-}" \
		--disable-profile \
		--without-gd \
		--without-cvs \
		--with-tls \
		libc_cv_ctors_header=yes \
		libc_cv_gcc_builtin_expect=yes \
		libc_cv_mips_tls=yes \
		libc_cv_forced_unwind=yes \
		libc_cv_c_cleanup=yes

	make install-headers install_root=$SYSROOT

	mkdir -p $SYSROOT/usr/lib
	make csu/subdir_lib
	cp csu/crt1.o csu/crti.o csu/crtn.o $SYSROOT/usr/lib

	if [ "$GLIBCVNO" == "2.15" ]; then # At least 2.19 does this with install-headers target it self.
		cp bits/stdio_lim.h $SYSROOT/usr/include/bits
	fi

	$TOOLS/bin/$TARGET-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o $SYSROOT/usr/lib/libc.so

	touch $SYSROOT/usr/include/gnu/stubs.h

	export LD_LIBRARY_PATH="$LD_LIBRARY_PATH_old"
}

phases+=(["5"]="gcc2")
phase_5() {
	log "Build bootstrapped gcc that can compile a full glibc."

	setup_and_enter_dir "$OBJ/gcc2"

	$SRC/$GCCV/configure \
		--prefix=$TOOLS \
		--target=$TARGET \
		--build=$BUILD \
		--host=$HOST \
		--with-sysroot=$SYSROOT \
		--with-pkgversion="${USER}'s $TARGET GCC phase2 cross-compiler" \
		--enable-languages=c \
		--disable-libssp \
		--disable-libgomp \
		--disable-libmudflap \
		--with-ppl=no \
		--with-isl=no \
		--with-cloog=no \
		--with-libelf=no \
		--disable-nls \
		--disable-multilib \
		--disable-libquadmath \
		--disable-libquadmath-support \
		--disable-libatomic \

	PATH="$TOOLS/bin:$PATH" make
	PATH="$TOOLS/bin:$PATH" make install
}

phases+=(["6"]="glibc full")
phase_6() {
	log "Building a full glibc for $TARGET."

	setup_and_enter_dir "$OBJ/glibc"

	LD_LIBRARY_PATH_old="$LD_LIBRARY_PATH"
	unset LD_LIBRARY_PATH

	local addons="--enable-add-ons"
	if glibc_needs_port_pkg; then
		addons="--enable-add-ons=nptl,ports"
	fi

	local extra_lib_cv=""
	if [ "$GLIBCVNO" == "2.15" ]; then # The one I've noticed, 2.19 does not need for example.
		extra_lib_cv="libc_cv_ctors_header=yes libc_cv_c_cleanup=yes"
	fi

	BUILD_CC=gcc \
	CC=$TOOLS/bin/$TARGET-gcc \
	CXX=$TOOLS/bin/$TARGET-g++ \
	AR=$TOOLS/bin/$TARGET-ar \
	RANLIB=$TOOLS/bin/$TARGET-ranlib \
	$SRC/$GLIBCV/configure \
		--prefix=/usr \
		--build=$BUILD \
		--host=$TARGET \
		--disable-profile \
		--without-gd \
		--without-cvs \
		$addons \
		--enable-kernel="${LINUXV##*-}" \
		libc_cv_forced_unwind=yes \
		$extra_lib_cv

	PATH="$TOOLS/bin:$PATH" make
	PATH="$TOOLS/bin:$PATH" make install install_root=$SYSROOT

	export LD_LIBRARY_PATH="$LD_LIBRARY_PATH_old"
}

phases+=(["7"]="gcc3")
phase_7() {
	log "Building the full GCC."

	setup_and_enter_dir "$OBJ/gcc3"

	$SRC/$GCCV/configure \
		--prefix=$TOOLS \
		--target=$TARGET \
		--build=$BUILD \
		--host=$HOST \
		--with-sysroot=$SYSROOT \
		--enable-languages=c,c++,go \
		--disable-libssp \
		--disable-libgomp \
		--disable-libmudflap \
		--disable-libquadmath \
		--disable-libquadmath-support \
		--with-pkgversion="${USER}'s $TARGET GCC phase3 cross-compiler" \
		--with-ppl=no \
		--with-isl=no \
		--with-cloog=no \
		--with-libelf=no

	PATH="$TOOLS/bin:$PATH" make
	PATH="$TOOLS/bin:$PATH" make install

	cd $TOOLS/bin
	for file in $(find . -type f); do
		tool_name=$(echo $file | sed -e "s/${TARGET}-\(.*\)$/\1/")
		ln -sf "$file" "$tool_name"
	done
}

phases+=(["8"]="testing")
phase_8() {
	log "Testing to compile a C program."

	test_path="/tmp/${TARGET}_test_$$"
	setup_and_enter_dir "$test_path"

	cat <<- EOF > helloc.c
	#include <stdlib.h>
	#include <stdio.h>

	int main(int argc, const char *argv[])
	{
		printf("%s\n", "Hello, MIPS world!");
		return EXIT_SUCCESS;
	}
	EOF

	PATH="$TOOLS/bin:$PATH" $TARGET-gcc -Wall -Werror -static -o helloc ./helloc.c
	log "RUN MANUALLY: Produced test-binary at: $test_path/helloc"


	log "Testing to compile a Go program."

	cat <<- EOF > hellogo.go
	package main

	import (
		"fmt"
	)

	func main() {
		fmt.Printf("%s\n", "Hello, Gopher!")
	}
	EOF

	# TODO enable when mgo is built
	#PATH="$TOOLS/bin:$PATH" go build -compiler gccgo ./hellogo.go
	#log "RUN MANUALLY: Produced test-binary at: $test_path/hellogo"

	log "Access compiler tools: $ export PATH=\"$TOOLS/bin:\$PATH\""
	log "Run dynamically linked Go programs: $ export LD_LIBRARY_PATH=\"$TOOLS/$TARGET/lib:\$LD_LIBRARY_PATH\""
}

list_phases() {
	echo "Available phases:"
	for phase_no in "${!phases[@]}"; do
		printf "\t%d => %s\n" "$phase_no" "${phases["$phase_no"]}"
	done
}


set +e
read -r -d '' help_text <<EOF
Erik Westrup's GCC cross-compiler builder

Usage: ${scriptname} -p phases | -l | (-h | -?)
	-p phases	The phases to run. Supported formats:
				1) 2	=> run phase 3
				2) 4-	=> run phase 4 to last (inclusive). "0-" => full build
				e) 1-5	=> run phases 1 to 5 (inclusive)
	-l			List available phases.
	-h, -? 		This help text.
EOF
set -e

phase_first=0
phase_last=8
phase_start="$phase_first"
phase_stop="$phase_last"


validate_phase() {
	local phase="$1"
	if ! ([ "$phase" -ge "$phase_first" ] && [ "$phase"  -le "$phase_last" ]); then
		printf "Invalid phase %d\n" "$phase"
		exit 2
	fi
}

parse_cmdline() {
	if [ "$#" -eq 0 ]; then
		echo "$help_text"
		exit 1
	fi
	while getopts "p:lh?" opt; do
		case "$opt" in
    		p)
    			if [[ $OPTARG =~ ^[[:digit:]]+$ ]]; then
    				phase_start="$OPTARG"
    				validate_phase "$phase_start"
    				phase_stop="$phase_start"
    			elif [[ $OPTARG =~ ^[[:digit:]]+-$ ]]; then
    				phase_start="${OPTARG:0:-1}"
    				validate_phase "$phase_start"
    				phase_stop="$phase_last"
    			elif [[ $OPTARG =~ ^[[:digit:]]+-[[:digit:]]+$ ]]; then
					IFS='-' read -a parts <<< "$OPTARG"
					phase_start=${parts[0]}
					phase_stop=${parts[1]}
    				validate_phase "$phase_start"
    				validate_phase "$phase_stop"
    				if [ "$phase_start" -gt "$phase_stop" ]; then
    					printf "invalid relation: %d <!= %d\n" "$phase_start" "$phase_stop"
    					exit 4
    				fi
    			else
    				echo "Bogus range." 1>&2
    				exit 3
    			fi
    			;;
    		l) list_phases; exit 0;;
    		:) echo "Option -$OPTARG requires an argument." >&2; exit 1;;
    		h|?|*) echo "$help_text"; exit 0;;
		esac
	done
	shift $(($OPTIND - 1))
}


parse_cmdline "$@"
for (( phase="$phase_start"; $phase <= "$phase_stop"; phase++ )); do
	log "$funcstars Stating phase $phase"
	log "$phasestars ${phases["$phase"]}"
	eval "phase_$phase"
	log "$phasestars ${phases["$phase"]}"
	log "$funcstars Completed phase $phase"
done
